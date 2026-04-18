### A. Describe how Vault Encrypts data
- In a sealed (encrypted) state, the only possible operations are unsealing the vault and checking the status of the seal. 
- Vault data is encrypted with the encryption key - an AES key.
- Vault data is sealed by default when the server starts. Vault knows where and how to access the physical storage, but doesn't know how to decrypt it. 
- In order to decrypt the data contained with vault, the vault needs to be unsealed - the plaintext root key to read the decryption key needs to be provided. 
- The encryption key used to encrypt the data in vault is stored in the keyring (part of the data). This key is encrypted with the "root key". The root key is stored alongside all other vault data, but is encrypted with the unseal key.
- The unseal key is encrypted by default with Shamir's Secret Sharing, which splits the root key into multiple shares. The unseal key can only be decrypted once a certain threshold of shares is provided; the process of providing these is known as the unseal process. 
### B. Explain how to seal and unseal Vault
- Unsealing can be done by running `vault operator unseal` or via the API. Each key can be entered from multiple mechanisms on any number of client machines.
- Once a node is unsealed, it remains unsealed until it's sealed by the API (which requires only a single operator with root privilege), the server is restarted or the storage layer encounters an unrecoverable error.

- Sealing is done with an API key, which throws away the root key and requires an unseal process to restore it.  This ensures that if an intrusion is detected the data in vault can be locked quickly and can't be accessed again without access to the root key shares..

- There also exists a method to reduce the operational complexity of keeping the unseal key secure - Auto Unseal. This delegates the responsibility of securing unseal keys from users to trusted devices or services. When Vault starts, it connects to the devices/services implementing the seal and ask it to decrypt the root vault key read from storage.
- There are other actions which require multiple users to complete, such as making a root token. When using the Shamir seal, the unseal keys are required. When using auto unseal, these need recovery keys. Initialisation with auto unseal generates recovery keys. In this situation, selection of recovery keys is automatic.
- Recovery keys CANNOT decrypt the root key, meaning that they cannot be used to unseal Vault if auto unseal does not work. There is a strict dependancy on the underlying seal mechanism, so if it becomes unavailable or deleted before the seal is migrated, access the vault cluster is lost until the mechanism is available again. 
	- IF THE SEAL MECHANISM OR ITS KEYS ARE DELETED, THE VAULT CLUSTER CANNOT BE RECOVERED.
	- Controls on the management of the seal mechanism are recommended to prevent disaster. Secondary clusters within the enterprise version can have a seal configured independantly of the primary, and when configured this can guard against the risk. Local mounts and other unreplicated items may still be lost.

- Recovery key SSS is configured based on the cli flags or api equivalents (replace - with \_) at `/sys/init`: 
	- recovery-shares - The number of shares to split the recovery key. 
	- recovery-threshold - The number of required to reconstruct the recovery key.
	- recovery-pgp-keys - The PGP keys to use to encrypt the recovery key shares.

- The unseal key can be rekeyed with a `vault operator rekey` operation, pending a threshold of keys. After rekeying, the new ke is wrapped by the HSM/KMS and stored like the previous key.  Note that seal wrapping requires 
- The recovery key can be rekeyed by adding `-target=recovery` to the previous operation. The API endpoint is `/sys/rekey-recovery-key` instead of `/sys/rekey`

- Seal migration is the process of migrating seals or seal types. This could include migrating between auto unseal and shamir seal. This cannot be done without downtime, requiring the whole cluster to be down briefly through the process. Seal migration requires both old and new seals be available. A backup should be taken before starting seal migration.

- Seal HA (high availability) allows more than one auto seal mechanism, meaning Vault can tolerate the temporary loss of a seal service or device for some time. A mix of seals should be selected to try and be sure that at least one seal is available at one time.
- If Seal HA configured at least 2 (with no more than 3 auto seals) vault can also start and unseal if only one seal is available, while vault will be in a degraded mode.
- When seals are added or removed, vault detects the need to rewrap CSPs and wrapped values and will begin the process. While a rewrap is in progress, changes to seal configuration is not allowed. 
- Seal HA can also be used to migrate between two auto seals in a simpler way.
- Shamir seals cannot be included Seal HA.
### C. Configure Environment Variables
- `VAULT_TOKEN` - A given vault authentication token - conceptually similar to a session token.
- `VAULT_ADDR` - The Vault's server as a url and port.
- `VAULT_CACERT` - Path to a PEM CA Cert File on the local disk to verify Vault's SSL cert. Higher priority than VAULT_CAPATH.
- `VAULT_CAPATH` - Path to a directory of PEM encoded CA files on the local disk to verify Vault's SSL cert.
- `VAULT_CLIENT_CERT` - Path to a PEM client cert used for TLS communication with the server.
- `VAULT_CLIENT_KEY` - Path to an unencrypted PEM private key corresponding to the VAULT_CLIENT_CERT.
- `VAULT_CLIENT_TIMEOUT` - Timeout variable. 60 secs by default.
- `VAULT_CLUSTER_ADDR` - Address used for other cluster members to connect to this node when in HA mode.
- `VAULT_FORMAT` - the vault output (read/write/status) in the given format. Either table, json or yaml.
- `VAULT_LICENSE` - A license to use with the node. Higher priority than VAULT_LICENSE_PATH and license path in config (Enterprise and server only).
- `VAULT_LICENSE_PATH` - A path to a license to use for the node. Higher priority than license path in config (Enterprise and server only).
- `VAULT_LOG_LEVEL` - Values are trace, debug, info (default), warn and err.
- `VAULT_MAX_RETRIES` - Max number of retries for some error codes. 2 by default (3 total attempts). Set to 0 for disabled retries. Applies to error codes 412 and 5xx except 501.
- `VAULT_REDIRECT_ADDR` - Address to be used when clients are redirected to this node in HA mode.
- `VAULT_SKIP_VERIFY` - Do not verify vault's certificate before communication. THIS IS NOT RECOMMENDED AND LEADS TO LOWER SECURITY.
- `VAULT_TLS_SERVER_NAME` - Name to use as the SNI host when communicating with TLS.
- `VAULT_CLI_NO_COLOR` - Removes ANSI colour escape characters in Vault outputs.
- `VAULT_RATE_LIMIT` - Most useful when using the client API (Go). Has the format rate\[:burst (optional, by default equals rate)]. Rate limiting is off by default.
- `VAULT_NAMESPACE` - The namespace to use for the command. Not necessary but allows for relative paths.
- `VAULT_SRV_LOOKUP` - Enables the client to lookup the host through DNS SRV lookup.
- `VAULT_MFA` - MFA credentials in the format `mfa_method_name[:key[=value]]`, where items in square brackets are optional. The CLI flag -mfa should be used instead if multiple credential values are needed (Enterprise only).
- `VAULT_HTTP_PROXY` - A HTTP/HTTPS proxy location which should be used by all requests to access vault. Format in URL form with port.
- `VAULT_PROXY_ADDR` - HTTP/HTTPS proxy location to be used by all requests to access vault. Format in URL with port. This will be prioritised over VAULT_HTTP_PROXY. When either of these are set, standard proxy variables HTTP_PROXY, HTTPS_PROXY and NO_PROXY will be ignored. The proxy applies to all requests.
- `VAULT_DISABLE_REDIRECT` - Prevents a client from following redirects. This may cause issues with commands like `vault operator raft snapshot` that redirect to the cluster's primary node.