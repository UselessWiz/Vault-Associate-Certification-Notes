### A. Explain Cluster Strategy for Self Managed and HC Managed clusters
- When using self-managed Vault, Cluster scaling is not built in and should be managed via other means - it is possible but not baked in.
- The self managed model means the client company is responsible for design, deployment, security, reliability, scaling and upgrading Vault clusters.
- The HCP dedicated service allows the client company to select a size and tier for the vault cluster, with the rest being handled by HCP.
- Both support DR and PR replication. 
- HCP enables auto-unseal by default, using a unique KMS key for each cluster, while SSS is the default on self managed. 
- Storage backends need to be chosen and managed by the client when self managing; HCP has integrated storage.
- Super User and top level access is root for the self managed version, and creating a root token is done when vault is initialized or requires unseal or recovery keys. For HCP, the top level namespace is admin, and an admin token can be generated in the portal with a lifetime of 6 hours.
- HCP clusters can be created on AWS or Azure across multiple regions. 
- HCP dedicated vault offers reduced operational over head, increased security, costs controlled, day zero readiness, reliability and ease of use.

Designing a cluster:
- The process involved planning out cluster architecture - determine storage needs, what needs to happen to run multiple clusters, determine replication needs and security needs (if zero trust is necessary).
- There's a number of best practices which can and should be followed when setting up a Vault cluster.
- Maintenance should also be considered - this is easiest done with Terraform and Sentinel. 

Examples of best practices:
- Adjust the default lease time (should be as low as possible).
- Use identity entities for accurate client counts (creating aliases to link each login to a single identity)
- Increase IOPs as necessary based on hardware and network. 
- Enable disaster recovery - enterprise provides a warm standby cluster with all primary cluster data, which can be promoted if needed.
- Create data snapshots periodically.
- Test data recovery by completing the failover and failback procedures.
- Establish an operating procedure for restoring a cluster from a snapshot.
- Have a suitable upgrade cadance - upgrade to the latest version of vault where possible.
- Test updates in a sandbox to check for compatibility issues without downtime or degraded service.
- Rotate audit device logs to prevent filesystem storage from being blocked.
- Monitor many metrics by default - audit logs, operational logs and telemetry data - ideally using a SIEM tool.
- Establish a usage baseline to check for abnormal activity.
- Avoid use of the root token - create ACL policies for required capabilities and revoke root tokens immediately after the action requiring it's use is done.
- Create a procedure for rekeying and perform as necessary to prevent loss of keys
### B. Explain the uses of storage backends
- Storage backends are used to represent the location of durable information in Vault. 
- They are configured through the Vault configuration file OR environment variables (which take priority).
- Integrated storage (RAFT) is recommended for most use cases - note that integrated storage was added in Vault 1.4. 
	- Integrated storage is supported by Hashicorp, requires no additional software on the server, is one less network hop (as opposed to jumping between vault and the external storage system), requires less troubleshooting if there's a problem (it's all contained to vault), encrypted data is stored on the same host, but is recommended to be used with SSDs.
	- Raft easily supports replication - one node's raft storage is selected as leader, with other nodes in the cluster having a replicated copy of the leader's data in their 'follower' node.
- HashiCorp Consul is recommended on Vault versions before 1.4 and is an alternative solution to data storage. Consul requires a Vault and Consul cluster, and the consul cluster should not be used for other data. All data is stored in memory. Snapshots should be taken more frequently and the max message size is smaller.
- Nodes can be joined to a cluster when using raft storage with the `vault operator raft join <leader_addr>` command on the follower node.
- This sets or needs the servers set to HA mode - servers in HA mode share the same storage. 
- Within the configuration file, a server can configure a `retry_join` stanza to automatically join the cluster like so:
```
storage "raft" {
   path    = "<path_to_local>/raft-vault_4/"
   node_id = "vault_4"
   retry_join {
      leader_api_addr = "http://127.0.0.2:8200"
   }
   retry_join {
      leader_api_addr = "http://127.0.0.3:8200"
   }
}
```
- Once a cluster is set up, only the leader node needs to be interacted with; the others will replicated the data.
- Snapshots can be taken with the command `vault operator raft snapshot save <snapshot_name>`. They can automatically be created by writing to the following endpoint, and taking the following data as an example:
```
vault write sys/storage/raft/snapshot-auto/config/daily interval="24h" retain=5 \
     path_prefix="raft-backup" storage_type="local" local_max_space=1073741824
```
- A snapshot can be recovered from with the command `vault operator raft snapshot restore <snapshot_name>`
- To change the leader, the `vault operator step-down` command can be used. Another node in the cluster (likely the first node added that isn't the node which stepped down) becomes leader.
- Cluster members can be removed with the command `vault operator raft remove_peer <peer_name>`.
- Nodes can be started in recovery mode for troubleshooting by passing the -recovery flag when starting the server. A recovery operational token can be generated which allows access to recovery functionality.
- When a node leaves recovery mode, it's list of cluster members is reset. To resume regular operations, each node needs to rejoin the cluster.
### C. Explain the uses of Shamir Secret Sharing and Unsealing
- The `vault operator rekey` command require a quorum of unseal keys, but can be used to change the number of total or threshold of key shares.
- The `vault operator rotate` command rotates the the underlying vault encryption key, and does not require downtime. It runs per cluster. Key rotation can be scheduled to run automatically by writing to `sys/rotate/config` with interval as data set to a time frame.

SSS is used to split the unseal key into parts. The unseal key is used to decrypt the vault encryption key.
### D. Explain the uses of disaster recovery and performance replication
- Replication requires enterprise. 
- The core unit of replication is a cluster, which is a collection of nodes - an active and corresponding HA nodes. All communication between nodes is end to end encrypted with mTLS setup with replication tokens exchanged during bootstrapping.
- Cluster replication also follows leader/follower - the leader cluster (primary) is linked to a series of follower secondary clusters. The primary cluster acts as the system of record and asynchronously replicates most vault data.
- Both forms of replication mirror the configuration of the primary cluster and the configuration of the primary cluster's backend. Disaster recovery mirrors tokens and leases for applications interacting with the primary cluster, and only the primary cluster is allowed to handle client requests. Performance replication allows secondary clusters to handle client requests, and they keep track of their own tokens and leases. When a secondary is promoted, apps must reauthenticate and obtain new leases.
- Data written to storage is either replicated (all downstream clusters receive), local (only downstream DR clusters receive) or ignored (not replicated).
- Performance replication makes it so secondaries keep track of their own tokens and leases, but share the underlying configuration and policies. If a user action modifies underlying shared state, the secondary forwards the request to the primary to the handled.
	- The primary cluster's mount configuration gets replicated across secondary clusters, but not all data needs to be replicated - eg. if primary cluster is in EU, and secondary clusters are outside the EU, GDPR requires PII not get transfered outside the EU unless sufficient levels of protection are in place. To comply, path filters can be set to allow or deny specific paths from being replicated.
- Clusters do not need to use the same storage engine or seal type, but note that replication will modify the seal on the secondary vault.
![[Pasted image 20260321182425.png]]
- It is possible to run different versions of vault between primary and secondaries, but its not recommended for longer than necessary. In order to set DR up, all clusters need to be on the same version.
- Some replication endpoints require sudo capability - generation of the secondary token. In particular, enabling DR is a very significant operation.
- There are endpoints that can be written to at `sys/replication` to control replication. Performance replication is handled by the `performance` endpoint under this. Replication can be enabled by writing to the `primary/enable` endpoint if this is a primary cluster, or `secondary/enable` for secondary clusters. To activate a secondary cluster, a token generated on the primary is required; this token can be requested by writing to the `sys/replication/performance/primary/secondary-token` endpoint with the `id=secondary`.
- It is critical that only one cluster is primary. This can only occur in situations where the original primary comes back online; to avoid this, the original primary can either be disabled, or the current primary must be demoted to a secondary.
- The process for enabling DR replication is the same as performance replication, replacing `performance` with `dr`.
- To promote a DR secondary cluster to be the primary, a DR operation token is required, which requires a threshold of unseal/recovery keys. To avoid situations where key holders can't coordinate to get their keys in a timely fashion, batch DR operation tokens can be created in advance.
	- These tokens need to be created on the primary cluster on a role with a policy that has update capabilities to `sys/replication/dr/secondary/promote`, `sys/replication/dr/secondary/update-primary` and (only if using raft - note this also need read capability) `sys/storage/raft/autopilot/state`
### E. Differentiate between Self Managed and HC Managed Vault clusters
- See A. It's extremely comprehensive.