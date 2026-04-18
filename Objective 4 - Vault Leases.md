All dynamic secrets and service tokens create a lease within Vault. This contains metadata such as time period, renewability, etc. Vault promises that data is valid for the given TTL, and once expired Vault can revoke the data (so the consumer of the secret is not certain of it's validity). This means comsumers of secrets need to check with vault to either renew the lease (if allowed) or request a replacement secret, making audit logs more valuable and key rolling easier. 
- Even if a dynamic secret is meant to be valid forever, it must have a lease to force the consumer to check in routinely.
- Leases can be revoked to invalidate the secret immediately and prevent further renewals. 
- The K/V backend does not issue leases, but sometimes returns a lease duration.
### A. Explain the purpose of a lease ID
- A lease ID can be used to renew and revoke a secret. It serves as an identifier for a given lease for other operations.
- A lease ID is returned when reading a dynamic secret (such as with the `vault read` command).
- Lease IDs are structured so that their prefix is always the path where the secret was requested from, allowing for the revocation of a tree of secrets.
- This means all secrets from a specific backend can be revoked quickly and easily if needed (such as in an intrusion).
### B. Describe how to renew leases
- Lease durations can be read - this is the TTL value. A lease must be renewed within its TTL.
- When renewing, a user can request a specific amount of time they want remaining on the lease, known as the increment. The increment is an increment from the current time, not from the end of the current TTL (i.e. if an increment value is 5 minutes, it's 5 minutes after the time the lease was renewed, not 5 minutes longer on top of the current TTL).
- Vault may not honour the requested increment - the backend in charge of the secret can choose to handle the increment however it likes, although generally it will respect the increment, limiting if to ensure the lease is renewed every so often.
- The command `vault lease renew [-increment=<time>] <lease-id>` is used to renew a given lease from it's ID.
### C. Describe how to revoke leases
- A specific lease can be revoked by it's ID with the following command: `vault lease [flags] revoke <lease-id>`. A force delete can be performed with the `-force` flag, meant for recovery situations where a secret was manually removed from the target secrets engine. The `-sync` flag can also be used to make the operation synchronous (do immediately i think) instead of queueing in the background.
- To revoke a prefix or a tree of leases, the command `vault lease revoke -prefix <path-prefix>` can be used. Note that the prefix is the path where the secret was requested from.
