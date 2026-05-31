# Amazon OpenSearch Service Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any OpenSearch issue.

## Guardrail 1: Cluster Health RED — Unassigned Primaries, Not Cluster Down
RED means at least one primary shard is unassigned. The cluster is NOT down — indices with all primaries assigned continue to serve reads and writes. Check `_cat/shards?v&h=index,shard,prirep,state,unassigned.reason` to identify which index and shard is affected. Common causes: insufficient disk space, node failure, or shard allocation filtering. Never tell users the entire cluster is unavailable when status is RED.

## Guardrail 2: JVM Memory Pressure Thresholds — 80% Is the Danger Zone
JVMMemoryPressure above 80% triggers aggressive garbage collection, causing latency spikes. Above 92%, the circuit breaker trips and rejects requests. Sustained pressure above 85% requires action: reduce field data cache, reduce query complexity, or scale the cluster. The JVM heap is set to half the instance RAM (max 32 GB due to compressed oops). Never recommend setting heap above 32 GB. Monitor both JVMMemoryPressure and OldGenJVMMemoryPressure.

## Guardrail 3: Shard Sizing — 10-50 GB per Shard, Max 1000 per Node
Each shard consumes heap memory, file handles, and cluster state overhead regardless of size. Undersized shards (< 1 GB) waste resources. Oversized shards (> 50 GB) slow recovery and search. AWS recommends 10-50 GB per shard. Keep total shards per node under 1000. Total shard count = primary_shards × (1 + number_of_replicas). Never recommend single-shard indices for large datasets or thousands of tiny shards.

## Guardrail 4: Dedicated Master Nodes — Always 3 or 5, Odd Number
Dedicated master nodes manage cluster state, shard allocation, and index operations. Without them, data nodes handle master duties under load, risking instability. Always deploy 3 or 5 dedicated masters (odd number prevents split brain). Master nodes do not hold data or serve search requests. For production workloads, dedicated masters are essential. Never recommend 2 or 4 master nodes — even numbers risk split brain.

## Guardrail 5: Storage Watermarks — 85% / 90% / 95% Are Hard Boundaries
OpenSearch enforces disk-based watermarks: 85% (low) stops allocating new shards to the node, 90% (high) relocates shards away from the node, 95% (flood stage) sets all indices on the node to read-only (`index.blocks.read_only_allow_delete`). After clearing space, you must manually remove the read-only block: `curl -XPUT "https://<endpoint>/_all/_settings" -H 'Content-Type: application/json' -d '{"index.blocks.read_only_allow_delete": null}'`. Never assume writes resume automatically after freeing space.

## Guardrail 6: UltraWarm and Cold Storage — Strictly Read-Only
UltraWarm and cold storage tiers are for read-only historical data. You cannot index, update, or delete documents in warm or cold indices. Data must be migrated from hot to warm using ISM policies or the `_ultrawarm/migration` API. Cold indices must be moved back to warm before they can be searched. Never suggest writing to warm or cold indices.

## Guardrail 7: Access Policy Evaluation — Resource Policy AND IAM Both Apply
OpenSearch domain access policies are resource-based policies (like S3 bucket policies). Access requires BOTH the resource policy AND the caller's IAM policy to allow the action. An explicit deny in either policy blocks access. For VPC domains, the resource policy still applies in addition to security groups. IP-based conditions in resource policies do not work for VPC domains (use security groups instead).

## Guardrail 8: VPC Domains Have No Public Endpoint — Cannot Change After Creation
Domains deployed in a VPC have no public endpoint and are only accessible from within the VPC or via VPN, VPC peering, Transit Gateway, or a reverse proxy. You cannot switch a domain between VPC and public access after creation — you must create a new domain and migrate data. Security groups control inbound access to VPC domains. Never suggest accessing a VPC domain from the public internet without a proxy or VPN.

## Guardrail 9: Serverless Uses Collections and OCUs — Not Domains and Nodes
OpenSearch Serverless is a fundamentally different architecture. It uses collections (not domains), OCUs (OpenSearch Compute Units) for capacity, and has separate APIs. Serverless manages shards, nodes, and scaling automatically. Data access policies, network policies, and encryption policies replace domain-level settings. Not all OpenSearch features are available in Serverless (e.g., no custom plugins, no ISM, no UltraWarm). Never apply managed domain troubleshooting (JVM, shards, nodes) to Serverless.

## Guardrail 10: FGAC Is Independent of Domain Access Policy
Fine-grained access control (FGAC) provides cluster-internal permissions (index-level, document-level, field-level). It uses an internal user database, SAML, or Amazon Cognito for authentication. FGAC is evaluated AFTER the domain access policy allows the request. Both layers must permit access. A common misconfiguration is allowing access in the resource policy but not mapping the IAM role to an FGAC backend role. Never assume resource policy access implies FGAC access.

## Guardrail 11: Bulk Indexing Best Practices — 5-15 MB per Request
The `_bulk` API is dramatically faster than individual document indexing due to reduced per-request overhead. Optimal bulk size is 5-15 MB per request. Too-large bulk requests (> 100 MB) cause JVM memory pressure and 429 (Too Many Requests) errors. Too-small bulk requests waste connection overhead. Use multiple threads for parallel bulk indexing. Monitor IndexingRate and 2xx/4xx/5xx metrics. Never recommend indexing documents one at a time for production workloads.

## Guardrail 12: Snapshot Repository Must Be Registered Before Use
Manual snapshots require registering an S3 repository using the `_snapshot/<repo>` API with an IAM role that has S3 and (optionally) KMS permissions. Automated snapshots are taken daily by AWS and stored internally — they cannot be restored to a different domain. For cross-domain restore or long-term retention, use manual snapshots to your own S3 bucket. The IAM role must be passed via the `iam:PassRole` permission. Never assume snapshots work without repository registration.
