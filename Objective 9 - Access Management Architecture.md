### A. Describe the Vault agent
- The Vault agent provides a simple and scalable way for apps to integrate with vault. The agent obtains secrets and passes them to the application. 
- This means that apps don't need to maintain vault integration code for all applications, or where there aren't enough resources to add such code to applications.
- Typical integration would require the use of the Vault API (Go) or the regular vault API to obtain and keep track of secrets. This is more complex. 
![[Pasted image 20260322154448.png]]
- Features of Agent:
	- Auto-auth (automatically authenticates with vault and manages the token renewal process for locally-retrieved dynamic secrets). This is configured with an auto-auth configuration stanza.
	- Client-side caching of newly created tokens and responses with leased secrets. 
	- Management of renewals of cached tokens and leases.
	- Runs as a windows service.
	- Can run as a child process with vault secrets injected as environment variables.
	- Use of Vault templates to render secrets to files or environment variables.
	- All controlled by a configuration file.
- Example config file:
```
pid_file = "./pidfile"

log_file = "/var/log/vault-agent.log"

vault {
  address = "https://vault-fqdn:8200"
  retry {
    num_retries = 5
  }
}

auto_auth {
  method "aws" {
    mount_path = "auth/aws-subaccount"
    config = {
      type = "iam"
      role = "foobar"
    }
  }

  sink "file" {
    config = {
      path = "/tmp/file-foo"
    }
  }

  sink "file" {
    wrap_ttl = "5m"
    aad_env_var = "TEST_AAD_ENV"
    dh_type = "curve25519"
    dh_path = "/tmp/file-foo-dhpath2"
    config = {
      path = "/tmp/file-bar"
    }
  }
}

cache {
  // An empty cache stanza still enables caching
}

template_config {
  static_secret_render_interval = "10m"
  exit_on_retry_failure = true
  max_connections_per_host = 20
}

template {
  source = "/etc/vault/server.key.ctmpl"
  destination = "/etc/vault/server.key"
}

template {
  source = "/etc/vault/server.crt.ctmpl"
  destination = "/etc/vault/server.crt"
}
```
- Defines templates at the bottom. Sinks allow access to files - in this case handling AzureAD .
- Auto auth config as shown - uses AWS. 
- The telemetry stanza can also be used to collect metrics like auth failures and successes, proxy request counts and cache hits and misses. 
- The listener stanza can be used enable API proxy - this is not recommended as Vault Proxy has replaced this functionality.
### B. Vault Secrets Operator
- Vault Secrets Operator (VSO) allows pods to consume vault secrets natively from kubernetes secrets. 
- It works by watching for changes to it's supported set of Custom Resource Definitions (CRD), with each CRD providing the specification required to allow synchronization from supported sources for secrets to kubernetes secrets. This is done by writing data to the destination kubernetes secret, so applications only need access to the destination secret to make use of the secret data. 
- Supported features:
	- Syncs from Vault or HCP Vault Secrets.
	- Automatic secret drift and remediation.
	- Automatic secret rotation for Deployment, ReplicaSet and StatefulSet kubernetes resource types.
	- Prometheus specific instrumentation for monitoring.
	- Support for Secret Data transformation.
	- Support for installation with Helm or Kustomize.
- By default, client cache does not persist, but the transit secrets engine can store and encrypt the client cache in the server. This is recommended if vault dynamic secrets is used.
	- This is done by configuring a policy to create and update endpoints within the transit secrets engine: `encrypt/vso-client-cache` and `decrypt/vso-client-cache`. A key also needs to be created by writing to `<transit-path>/keys/vso-client-cache`.
	- A kubernetes role also needs to be created in Vault like so:
	```
vault write auth/<VAULT_KUBERNETES_PATH>/role/operator \
bound_service_account_names=vault-secrets-operator-controller-manager \
bound_service_account_namespaces=vault-secrets-operator \
token_period="24h" \
token_policies=operator \
audience="vault"
	```
	- The vault connection then needs to be set up between VSO and the Vault server; the schema for this is as demonstrated below:
	```
	apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: local-vault-server
  namespace: vault-secrets-operator
spec:
  address: 'https://vault.vault.svc.cluster.local:8200'
	```
	- Then, storage encryption on the client cache can be enabled.
- VSO can also get instant updates for static secrets from vault by subscribing to event notifications from vault.
	- The VaultAuth role needs to have subscribe capability to the required secrets and needs to have read access to `sys/events/subscribe/<secret_engine>`. 
	- Then, enable syncConfig.instantUpdates=true within the VSO config under the vaultstaticsecrets spec.
	- Success can be confirmed by checking kubernetes events on the VaultStaticSecret resource.
- Data can be transformed into a format compatible wiht the application, specified with either a secret custom resource (CR) or references to SecretTransformation CR instances (or both).
	- This is done with the templates for Golang, same as used by Vault agent for the same purpose. 
	- Secret data can be accessed via the .Secrets input member; to include a password in the application's secrets a template can be specified like so: `{{- printf "password=%s" (get .Secrets "password") -}}`
	- Secret metadata can also be accessed with the .Metadata input member: `{{- printf "secretGroup=%s" (get .Metadata "secretGroup") -}}`
	- Resource annotations can be accessed with the .Annotations input member, and consists of all metadata.annotations on the secret custom resource: `{{- printf "host=%s" (get .Annotations "myapp.config/host") -}}`
	- Resource labels can be accessed with the .Labels input member, and includes all metadata.labels on the secret custom resource `{{- printf "appType=%s" (get .Labels "appType") -}}`
	- There's a handful of template functions which can perform functions like trim (remove whitespace), b64enc and b64dec (base 64 encode and decode), get (retrieves value from map input) and dig (retrieve specific value or return a default if keys are not found).
	- Transformations can be set up locally for a singular secret resource, such as syncing dynamic Postgres DB credentials to a kubernetes secret or shared by various secret CRs (where those CRs can reference shared secret trasnformations).