### A. Define the purpose of authentication methods.
- Authentication allows a user, either human or otherwise, to access the resources they need to in order to complete tasks. An authentication method is how a user proves they are who they say they are, and therefore that they have permission to access what they're requesting; this could be through something they know (a password), something they have (a physical security key) or something they are (a biometric indicator like a fingerprint). 
- Within Vault, different authentication methods are enabled through different providers. They are enabled through vault **plugins**. They allow for the assignment of identities and policies to a user to configure what they're allow to access within the vault, etc.

### B. Choose an Authentication method based on use case.
- Humans should choose human auth methods. Machines should chose system authentication methods.
- See below.

### C. Explain the difference between human and system authentication methods.
- A human authentication method allows a human user to authenticate with a platform, including systems such as GitHub, LDAP or username and password.
- Within vault, this is typically handled by an externally configured authentication provider, such as AWS, Github or Kubernetes - SSO ideas.

- Machine authentication methods are designed to allow automatic, machine driven systems to authenticate with and use Vault capabilities without the need for human intervention, and includes methods like AppRole, AWS, Kubernetes and TLS.

### D. Define the purpose of Identities and Groups.
- Identities include entities, aliases and groups.
- Entities are the representation of a single user within Vault. 
- One entity in Vault can be represented by multiple identities from external identity providers; these are represented by aliases of an entity - Each entity is made up of zero or more aliases. ![[Pasted image 20260305213837.png]]
- One identity cannot have multiple aliases from the same auth mount. It can have multiple aliases with the same auth type (GitHub), as long as the mount point is different (different tenants for example). 
- Local auth methods are also supported.
- Entities do not automatically pull identity information from anywhere, they need to be managed explicitly. 
- Vault identities are a cache of identities, not a source of identities.
- Each entity is tied to the alias token.
- Policies provide a way to access specific paths and operations, and can be defined at an entity or alias level. Policies on the entity grant additional capabilities to the policies granted by alias tokens.
- Vault will create an entity when a user first logs in, but entities can also be created by operators before a user logs in. 
- Groups contain multiple entities as members, and can have sub groups. Policies can be applied to groups, which will apply to all teh entities within the group. 
- Groups like this are internal groups. Membership management is conducted manually in an internal group. External groups are mapped to groups outside of Vault. External groups can have only one alias, mapping to the notion of a group that is outside the identity store.
>For example, groups in LDAP and teams in GitHub. A username in LDAP belonging to a group in LDAP can get its entity ID added as a member of a group in Vault automatically during _logins_ and _token renewals_. This works only if the group in Vault is an external group and has an alias that maps to the group in LDAP.

### E. Authenticate to Vault using the API, CLI and UI.
- CLI: Logins are done with the `vault login` method, supporting many built-in auth methods. The command outputs the raw token.
- The API is usually used for machine identities. It's is a POST request to 
  `$VAULT_ADDR/v1/auth/userpass/login/{vault-user}`, with data in the form of a JSON string containing the key "password" and the value of the password for that user (assuming local auth I believe). This returns a token as the value for key auth.client_token. This token has the permissions granted by the policies for that entity and alias for that login method. This token can then be used to interact with other Vault APIs like in the following example which creates a secret called "creds" with key password and value as below: ```curl -s \
  -H "X-Vault-Token: $DANIELLE_DEV_TOKEN" \
  -X PUT \
  -d '{ "data": {"password": "Driven Siberian Pantyhose Equinox"} }' \
  $VAULT_ADDR/v1/dev-secrets/data/creds```
- Loging in through the UI is done on the vault login page. Users can select whichever method they wish to authenticate.

### F. Configure Authentication methods using the API, CLI and UI.
- To set up an auth method, it must first be set up. 
	- CLI: vault auth enable <auth_method_type>
	- API: POST to endpoint `$VAULT_ADDR/v1/sys/auth/userpass`, data in the form `{"type": <auth_method_type>}`
	- UI: Main Navigation -> Access -> Enable new auth method ->Path: <auth_method_type>. 
- When using the userpass method, Vault can attach ACL policies to tokens that are issued, which controls what paths can be accessed by the token and what capabilities it has. These are written in HashiCorp Configuration Language (HCL):```
path "dev-secrets/+/creds" {
  capabilities = ["create", "list", "read", "update"]
}
- These polices are activated in different ways:
	- CLI: vault policy write <policy_name>\n\<insert policy>
	- API: PUT at endpoint `$VAULT_ADDR/v1/sys/policies/acl/developer-vault-policy` with data `{"policy": <insert policy>}`
	- UI: Main Navigation -> Create ACL policy -> enter name -> enter policy into text box -> hit create policy.
- Users can then be created (or depending on auth method, set up with a connection to an auth backend). For userpass:
	- CLI: vault write <path_to_user (username)> password=<user_password> policies=\<insert policies>
	- API: POST to <path_to_user (username)> with data `{"password":<password>, "token_policies": <insert policies>}`.
	- UI: View userpass method -> Create user -> enter username and password -> expand tokens and scroll to policies section -> add desired policies -> save.
- A secrets engine can then be made to allow secrets to be created and stored. Vault has some enabled by default. A new one can be made with the following.
	- CLI: vault secrets enable -path=\<path>\<type>
	- API: POST to `$VAULT_ADDR/v1/sys/mounts/<path>` with data `{"type":<type>`}. Response is a HTTP 204 status with no response data.
	- UI: Main Navigation ->Secrets engine -> Enable new engine -> select type -> enter path -> configure options for selected plugin.
- Once this is done, logging in is possible.
