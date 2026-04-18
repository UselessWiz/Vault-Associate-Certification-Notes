### A. Choose between Service and Batch tokens based on use case
- Service tokens are 'normal' tokens which support all features (renewal, revocation, creating child tokens, etc). This means they are heavy to create and track within vault.
- Batch tokens are encrypted Binary Large OBjects (BLOBs) which carry enough information to be used for some vault actions without using storage space. They are lightweight but don't have as much functionality or flexibility as service tokens.

|                                                     | Service Tokens                                                                               | Batch Tokens                                                                                                         |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Can be Root Tokens                                  | Yes                                                                                          | No                                                                                                                   |
| Can Create Child Tokens                             | Yes                                                                                          | No                                                                                                                   |
| Can be Renewable                                    | Yes                                                                                          | No                                                                                                                   |
| Manually Revocable                                  | Yes                                                                                          | No                                                                                                                   |
| Can be Periodic                                     | Yes                                                                                          | No                                                                                                                   |
| Can have Explicit Max TTL                           | Yes                                                                                          | No (always uses a fixed TTL)                                                                                         |
| Has Accessors                                       | Yes                                                                                          | No                                                                                                                   |
| Has Cubbyhole                                       | Yes                                                                                          | No                                                                                                                   |
| Revoked with parent (if not orphan)                 | Yes                                                                                          | Stops working                                                                                                        |
| Dynamic Secrets lease assignment                    | Self                                                                                         | Parent (if not orphan)                                                                                               |
| Can be used across performance replication clusters | No                                                                                           | Yes (if orphan)                                                                                                      |
| Creation scales with performance standby node count | No                                                                                           | Yes                                                                                                                  |
| Cost                                                | Heavyweight; multiple storage writes per token creation                                      | Lightweight; no storage cost for token creation                                                                      |
| Lease Handling                                      | Leases created are tracked along with the service token, and revoked when the token expires. | Leases created are constrained to the remaining TTL of the batch token and, if not an orphan, tracked by the parent. |
Batch token leases are revoked when the batch token's TTL expires or when the batch token's parent is revoked - the batch token is also denied access to the vault. Batch tokens can be used across performance replication clusters, but only if they're an orphan.
### B. Describe root token uses and lifecycle
- Root tokens have the root policy attached - this means they can do anything. They are the only token which can be set to never expire without renewal.
- Root tokens can be created in three ways: 
	- The initial root token when the vault is initiated (which has no expiration and should be manually revoked).
	- Using another root token; note that if a root token has an expiration, it cannot be used to make a root token without expiration.
	- Using the `vault operator generate-root` command with the permission of enough unseal key holders. The `--init` flag must be passed to initialise root token generation; running this will generate a nonce used to identify the token generation attempt, and either a OTP or a PGP key which can be used to decode the resulting encoded token once the generation process is complete. Unseal key holders can then use the same command (without any flags) and pass the nonce and their unseal key to generate a root token.
- It's recommended to use root tokens only for initial setup (setting up auth methods and policies so admins can use more limited tokens) or in emergency situations. They should be revoked as soon as they're not needed.
- When a root token is in use, it's recommended to have multiple people watching to verify the tasks performed and to ensure the token is revoked once done.
### C. Explain the purpose of token accessors
- The accessor is a value which references a token and can be used to look up the token's properties, view the token's capabilities on a path, renew the token or revoke it (note that to use these functions, the token making the call requires permissions for the functions, not the token associated with the accessor - i.e. **you** need to have permissions to act on another person's token).
- Token accessors can be used in a variety of different ways - a service that creates tokens can store the accessor related to a job, then use the accessor to revoke the token once the job is done. If the audit log is set to not obscure token accessors, malicious tokens can be quickly revoked in an emergency (although audit logs can be used to perform larger DDoS attacks in this case).
- Tokens can only be listed by accessors - the `auth/token/accessors` command.
### D. Explain the impact of time-to-live
- The TTL is a current period of validity since the token's creation or last renewal (whichever is more recent). Root tokens can have a TTL as well, but some will also be set to 0 to indicate that the token won't expire.
- Once the TTL is up, the token will no longer function.
- If a token can be renewed, vault can extend the token with the command `vault token renew`
- Periodic tokens are a special type of service token which have a TTL, but no max TTL; this means the same token can live indefinitely as long as they're renewed within their TTL period. If the token is not renewed before it's TTL is reached the token becomes invalid.
- The default TTL is 32 days, but can be overridden globally or per auth method. 
### E. Explain Orphaned tokens
- Orphan tokens have no parent - they are the root of their own token tree. 
- Normally when a token holder creates new tokens, they're created as children of the original token. When a parent token is revoked, all the children and their leases are revoked as well, ensuring that a user cannot escape revocation by creating a never-ending tree of child tokens.
- Users with appropriate permissions can revoke orphan tokens using the `auth/token/revoke-orphan` endpoint, which revokes the orphan token but sets the token's immediate children to be orphans. This means that the rest of the tree remains valid - use with caution.
### F. Describe how to create tokens based on need
- Tokens can be used directly, or auth methods can be used to dynamically generate tokens based on external identities.
- A client must first authenticate with vault before receiving a token.
![[Pasted image 20260309195834.png]]
- Regular tokens can also be created with the `vault token create -policy=<policy>` command. A period can be specified.
- To create an orphan token:
	- Write access to the `auth/token/create-orphan`
	- Having sudo or root access to the `auth/token/create` endpoint and setting the `no_parent` parameter to true.
	- Token store roles
	- Logging in with any other non-token auth method.