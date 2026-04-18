### A. Explain the value of Vault Policies
- Policies are a way to declaratively grant or forbid access to operations and resources in Vault. As everything in Vault is a path, these policies control operations that apply to specific paths.
- Policies are applied to tokens once they're authenticated. It is important to bear in mind that once a token is issued, the policies it possesses cannot be changed. If the effective policies of a token needs to be updated, the token must be revoked and the user will need to reauthenticate to receive the updates. IF THE CONTENT OF THE POLICY CHANGES, THE TOKEN WILL UPHOLD THESE CHANGES.
- Policies are stored on disk, and their name acts as a pointer to it's set of rules (policies are referenced by name). Data in the auth method to a policy, for example, members of the group "dev" map to the vault policy named "readonly-dev".
![[Pasted image 20260307180333.png]]
- Policies can also specify allowed parameters, required parameters and denied parameters at a path to constrain requests, see the following snippets:
```python
# Policies can also specify allowed, disallowed, and required parameters. here the key "secret/restricted" can only contain "foo" (any value) and "bar" (one of "zip" or "zap"). If a parameter "*" is set to an empty array ("*" = []), all other parameters can be modified with other restrictions remaining.
path "secret/restricted" {
  capabilities = ["create"]
  allowed_parameters = {
    "foo" = []
    "bar" = ["zip", "zap"]
  }
}

# This requires the user to create "secret/profile" with a parameter/key named "name" and "id" where kv v1 is enabled at "secret/".
path "secret/profile" {
  capabilities = ["create"]
  required_parameters = ["name", "id"]
}

# This allows the user to update the userpass auth method's user configurations (e.g., "password") but cannot update the "token_policies" and prevents modifiying the "admin" policy.
# Denied parameters take priority over allowed parameters.
# If "*" = [] is set, all parameters will be denied.
path "auth/userpass/users/*" {
  capabilities = ["update"]
  denied_parameters = {
    "token_policies" = []
    "policies" = ["admin"]
  }
```
- All parameter values support globbing at the start and end.
- Note that policies are evaluated without considering default values - if a denied parameter prevents a bool from being considered false and it is not included in the operation (i.e. set to false by default) the operation will be permitted. This situation can be prevented by making the parameter required as well, see below:
```python
path "secret/foo" {
  capabilities = ["create"]
  required_parameters = ["no_store"]
  denied_parameters = {
    "no_store" = [false, "false"]
  }
}
```

Notation
- HCL is very similar to JSON.
- Comments are done with a # at the start of a line and is used to troubleshoot or aid in understanding of patterns or why a policy exists.
### EXAMPLE POLICY USED BY THE FOLLOWING
```python
path "secret/data/creds" {
  capabilities = ["create", "update"]
}

path "secret/data/creds/confidential" {
  capabilities = ["read"]
}
```
### B. Describe Vault Policy: Path
Path describes which path within Vault the policy applies to. Policies apply to all sub-paths specified by a given path: i.e. a policy for `secrets/data` will also apply to `secrets/data/credentials`. A separate path for `secrets/data/credentials` can be defined to replace the capabilities at this path.
- More granular control over the path can be achieved using the glob pattern (\*). Note that this is only supported at the end of a path.
- The + wildcard can be used to match anything, for example `secrets/+/credentials` will allow access to any paths that have a credentials sub-path after the secrets endpoint (in the above, + will match `data`, but also match `other-data` and any other paths that might exist under the secrets endpoint.). 
- Paths can also use templates to restrict access based on identity, such as in the following path which enables access to only a user's specific credentials based on their name: `secrets/+/credentials/{{identity.entity.name}}`. This allows POLP to be followed within a single policy.
- The following templates exist:

|Name|Description|
|:--|:--|
|`identity.entity.id`|The entity's ID|
|`identity.entity.name`|The entity's name|
|`identity.entity.metadata.<metadata key>`|Metadata associated with the entity for the given key|
|`identity.entity.aliases.<mount accessor>.id`|Entity alias ID for the given mount|
|`identity.entity.aliases.<mount accessor>.name`|Entity alias name for the given mount|
|`identity.entity.aliases.<mount accessor>.metadata.<metadata key>`|Metadata associated with the alias for the given mount and metadata key|
|`identity.entity.aliases.<mount accessor>.custom_metadata.<custom_metadata key>`|Custom metadata associated with the alias for the given mount and custom metadata key|
|`identity.groups.ids.<group id>.name`|The group name for the given group ID|
|`identity.groups.names.<group name>.id`|The group ID for the given group name|
|`identity.groups.ids.<group id>.metadata.<metadata key>`|Metadata associated with the group for the given key|
|`identity.groups.names.<group name>.metadata.<metadata key>`|Metadata associated with the group for the given key|
### C. Describe Vault Policy: Capabilities
Capabilities describe the actions that this policy permits or denies at the specified path. They are defined as a JSON array of strings.
- Common capabilities are create, list, read, update and deny.
- When there is a conflict with the capabilities (i.e. the same path is referenced by the policy), the most specific path takes precedence. Additionally, the deny capability is higher priority than all others. If paths are identically defined and there is no deny capability, the capabilities are merged.
- Other possible capabilities and their rough mapping to HTTP verbs where applicable are shown below. Note that the mapping between Vault actions and HTTP verbs is not 1:1, but they are very similar.
	- create (POST/PUT) - Allows creating data at the given path. Very few parts of Vault distinguish between create and update, so most operations require both create and update capabilities. Parts of Vault that provide such a distinction are noted in documentation.
	- read (GET) - Allows reading the data at the given path.
	- update (POST/PUT) - Allows changing the data at the given path. In most parts of Vault, this implicitly includes the ability to create the initial value at the path.
	- patch (PATCH) - Allows partial updates to the data at a given path.
	- delete (DELETE) - Allows deleting the data at the given path.
	- list (LIST) - Allows listing values at the given path. Note that the keys returned by a list operation are not filtered by policies. Do not encode sensitive information in key names. Not all backends support listing.
	- sudo - Allows access to paths that are root-protected. Tokens are not permitted to interact with these paths unless they have the sudo capability (in addition to the other necessary capabilities for performing an operation against that path, such as read or delete). For example, modifying the audit log backends requires a token with sudo privileges.
	- deny - Disallows access. This always takes precedence regardless of any other defined capabilities, including sudo.
	- subscribe - Allows subscribing to events (arbitrary, non-secret data exchanged between vault/plugins and vault components/external users) for the given path.

### D. Choose a Vault Policy based on Requirements
- The most specifically defined policy has the highest priority - the most specific path takes priority. In this situation, permissions are not inherited, so if `secrets/data` provides create, read, list and update privileges, but `secrets/data/credentials` has capabilities of read, this more specific path will ONLY have read capability.
- Paths deny by default (so if a path is not specified, it will be denied). Path can also be explicitly denied if needed, for example in the below policy to restrict access to root credentials.
```python
# Vault policy to allow access to the dev-secrets k/v v2 secrets engine
path "dev-secrets/+/*" {
  capabilities = ["create", "list", "read", "update"]
}

path "dev-secrets/+/root" {
  capabilities = ["deny"]
}
```
- There are two default policies: default and root. The default policy provides basic functionality for a token to look up data about itself, and to use cubbyhole data related to it. This policy doesn't need to be included on tokens if they are created with the --no-default-policy flag (or equivalent within the API). The root policy allows access to ANYTHING in vault, and should NOT be included
### E. Configure Vault Policies using the UI and CLI
CLI
- `vault read sys/policy OR vault policy` will list all registered policies.
- `vault policy write policy-name policy-file.hcl` creates a policy, essentially creating a symlink between the name of the policy and the policy ACLs
- `vault write sys/policy/existing-policy policy=updated-policy.json` updates a policy with the provided json.
- `vault delete sys/policy/policy-name` deletes the specified policy.

When a user is created via the CLI, policies can be applied like so, such that whenever this user authenticates, the token generated : `vault write auth/userpass/users/{user} password="s3cr3t!" policies="dev-readonly,logs"`

UI
- Navigate to the dashboard and hit policies -> Create ACL Policy.
- Enter a name and write/copy and paste your policy in.
- Hit "Create Policy".
- Once a previously created policy is created, you can edit it from this screen as well.
- The Policies screen will list UI policies.

The UI needs to be activated first - NOTE THAT YOU CANNOT MAKE POLICY ADJUSTMENTS TO THE `ui/mounts` AND `ui/resultant-acl` ENDPOINTS ONCE THE VAULT UI IS ENABLED.

Depending on configuration, UI policies may need to be defined with different ACL capabilities. The default UI policy includes the paths above and cannot be modified with additional paths once the UI is enabled.
- `/sys/internal/ui/mounts` - provides a list of currently visible mounts based on the listing_visibility parameter. `sys/internal/ui/mounts` is an unauthenticated, internal endpoint used for UI and CLI preflight checks. Requests that include an X-Vault-Token will return all mounts the token has path capabilities on.
- `/sys/internal/ui/resultant-acl` - repackages authentication information used by the UI. If you do not have have permission to call the `ui/resultant-acl` endpoint, you may receive warnings or errors in the UI. **THIS PATH IS CRITICAL WHEN CONFIGURING THE UI**


