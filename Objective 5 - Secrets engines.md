### A. Choose a secrets engine based on use case
-  Static secrets do not expire, and the process of changing such secrets is manual. They are rarely rotated. Examples: 3rd party tokens, API keys, PKI certificates, PGP keys, encryption keys, usernames and passwords. The KV engine store and manages static secrets, and is the most commonly used engine for such secrets.
- Dynamic secrets are created on demand and revoked after a predetermined amount of time (TTL). Dynamic secrets do not exist until a request is made to create one. They are not held in storage. Revocation occurs based on the secrets engine configuration, limiting the time it's available. Secrets are typically created through an integration with a third party platform. Examples: Database keys, cloud provider credentials or any other short lived secrets. 
- Secrets engines can also handle data in transit. Instead of storing data, the engine handles encryption and decryption of data.
### B. Compare and contrast dynamic secrets vs. static secrets (use case, etc)
- Dynamic secrets can be used to connect to databases by generating a dynamic secret connected to a database instance.
	- Vault uses it's connection to create a username and password within the database in SQL - this is then passed to the database when a secret is requested and the result is passed to the user.
	- This is done by writing a configuration to a specified endpoint. The request is made by reading from the endpoint.
- Static secrets are stored securely and read upon request, and do not change often.

The Database Secrets Engine:
- Vault supports static roles for database secrets engines, which are a 1-1 mapping of vault roles to database usernames. This means that vault stores and rotates passwords for the database user based on a configurable rotation schedule.
- When a client requests credentials for the static role, Vault returns the current password for that database user.
- Note that static roles should not be used for root database credentials. This is designed for other static database users.
- Configurations can be set in the `database/config` path, including setting a rotation schedule (cron job for when rotations of the root credential should be done) and a rotation window (how long to try rotating for):
```
vault write database/config/my-mssql-database \
...
rotation_window="1h" \
rotation_schedule="0 * * * SAT"
...
```
- Rotation can also be disabled by setting `disable_automated_rotation` to true. 
- NOTE: Schedule based rotation is an Enterprise only feature.
### C. Describe the use of transit secrets engines
- Transit secrets engines do not store data sent to it. Instead it encrypts or decrypts data sent to it. It can also sign and verify data, generate hashes and HMACs and act as a source of randomness.
- This engine means that encryption and decryption doesn't need to be handled by application developers - the responsibility falls on Vault.
- Supported key types include AES128-GCM96, AES256-GCM96, ED25519, ECDSA-P256, ECDSA-P384, ECDSA-P521, RSA-2048, RSA-3072, RSA-4096, CHACHA20-POLY1305.
- Keys can be versioned to ensure both speed when retrieving keys and also security to ensure keys aren't used excessively. Keys older than min_decryption_version are archived, but can be retrieved in emergencies.
- Convergent encryption (plaintext + context = same ciphertext) is supported through derivation of a key and deterministic nonce. This has uses such as database storage with limited lookup capabilities. There are a few supported versions which have been historically supported.
- The transit secret engine doesn't require configuration, it just needs to be enabled: `vault secrets enable transit`. By default it enables at `transit/`
- Encrypting plaintext data is done by writing to the `transit/encrypt` endpoint and passing plaintext. The returned ciphertext starts with "vault:v1:" to indicate that it's been wrapped with vault and the key version used. Plaintext MUST be base64 encoded, and the remaining ciphertext is a concatination of the IV and ciphertext.
### D. Describe the purpose of secrets engines
- Secrets engines are components which store, generate or encrypt data. They take a provided input, process that input and return an output.
	- Some simply store and read data (encrypted redis/memcached).
	- Others connect to other services and generate dynamic credentials as needed.
	- Some provide encryption as a service, TOTP generation, certificates and more.
- They're designed to be flexible.
- They exist at a path in vault. When a request comes to vault, the router routes anything with the route's path prefix to the secrets engine - each engine defines it's own path and properties.
- For users, secret engines act similarly to a virtual file system with read, write and delete operations.
- Secrets engines can be enabled at a given path, where each engine is isolated to it's path. By default, they're enabled at their type ("aws" enables at `aws/`).
- They can be disabled. Note that when a secrets engine is disabled, all secrets are revoked (if supported) and all data stored in the physical storage layer is deleted.
- The path for a secrets engine can be moved, which revokes all secrets as leases are tied to the path they were created. Configuration data persists.
- Global configuration can also be tuned - TTLs for example.
### E. Describe the use of response wrapping
- Response wrapping minimizes the chance of a secret being transmitted in plaintext, for example if a machine needs to get a TLS private key where it doesn't store one permanently. 
- To resolve this, when a response would be sent to a HTTP client, it can instead insert it into the cubbyhole of a single use token, providing that token instead. 
- The response is wrapped by the token, retrieving it requires an unwrap operation against the token. 
![[Pasted image 20260314135325.png]]
- This provides confidentiality by ensuring the value transmitted is only a reference to the secret, meaning logs captured don't directly see the sensitive information.
- It provides malfesance detection by ensuring only a single party can unwrap the token - a client receiving a token that cannot be unwrapped can trigger a security incident. Clients can also validate that a given token is from an expected location in vault.
- It limits the lifetime of secret exposure as the token typically has a short lifetime. It is always separate from the wrapped secret.
- Response wrapping is done per token. If a client requests wrapping, the HTTP repsonse is serialized, a new token is generated with the provided TTL, The original response is stored in the token's cubbyhole and a new response is generated with the wrap information. The response is then returned to the caller.
- Validation should be done by ensuring that the response arrives in due time (if it doesn't an attacker may have intercepted it and should trigger an alert for immediate investigation), performing a lookup on the token to see if it's been unwrapped, expired or revoked. If the token is invalid, it's likely been intercepted and should alert for immediate investigation. Once the token informaiton is in hand and confirmed correct, validate taht the creation path is as expected - this should be as specific as possible. After this, unwrap the token an d investigate immediately if it fails.
### F. Explain the value of short-lived, dynamic secrets
- The main value of short-lived dynamic secrets is that short-lived secrets are much more unlikely to be cracked
	- If a secret only lasts 10 minutes, that gives an attacker only 10 minutes to try and break the password. Even if an attacker is able to get a secret, they'll only have access to a service for a short period of time, limiting the level of damage they can hopefully perform.
- It also allows for better auditing as each service accesses the database with unique credentials.
- Key rotations are logged in the vault.log on both successful and failed rotations to highlight any issues and to confirm progress; this serves as further auditing for this process.
### G. Enable secrets engines using the CLI, API and UI
CLI:
- `vault secrets enable <engine>` enables the secrets engine. The `-path` argument can be specified to enable at a different path.
API:
- The API makes you specify your endpoint which is where the secrets engine will be mounted. The data for the request to create the secrets engine is done with a JSON payload containing the "type" of secrets engine:
```
curl \
    -H "X-Vault-Token: $VAULT_TOKEN" \
    -X POST \
    -d '{ "type":"kv-v2" }' \
    $VAULT_ADDR/v1/sys/mounts/dev-secrets
```
GUI:
- This is done from the "Secrets Engine page", selecting create new engine and selecting a type. You can give it a path and change other settings from the wizard window that appears. Once confirming all options are right., you can select "enable engine" and the engine is made. 
### H. Access vault secrets using the CLI, API and UI
- Reading from an endpoint with a secret will retrieve the secret, whether this be `vault read <endpoint>` or a GET call at the endpoint with a secret.