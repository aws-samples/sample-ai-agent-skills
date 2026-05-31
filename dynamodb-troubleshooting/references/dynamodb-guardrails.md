# DynamoDB Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any DynamoDB issue.

## Guardrail 1: Throttling is Per-Partition
Provisioned capacity is distributed across partitions. A table with 1000 WCU and 10 partitions gives ~100 WCU per partition. A hot partition throttles even with available table-level capacity.

## Guardrail 2: GSI Throttling Causes Base Table Throttling
If a GSI is throttled on writes, it backpressures the base table. The base table write is rejected even if the base table has capacity. Always check GSI capacity when diagnosing write throttling.

## Guardrail 3: On-Demand Has Scaling Limits
On-demand mode scales based on previous peak traffic. It can handle up to 2x the previous peak instantly. Traffic beyond that may be throttled until DynamoDB adapts (minutes).

## Guardrail 4: Scan Consumes Full Table Capacity
Scan reads every item in the table regardless of filter expressions. Filters reduce the result set but NOT the consumed capacity. A filtered scan on a 10 GB table consumes capacity for 10 GB.

## Guardrail 5: TTL Deletion is Best-Effort
TTL-expired items may take up to 48 hours to be deleted. They still appear in queries and scans until deleted. Filter them in your application using the TTL attribute.

## Guardrail 6: Transactions Cost 2x
Transactional reads and writes consume twice the capacity of non-transactional operations. TransactWriteItems costs 2 WCU per 1 KB. TransactGetItems costs 2 RCU per 4 KB.

## Guardrail 7: Item Size Limit is 400 KB
Each item (including all attribute names and values) cannot exceed 400 KB. This is a hard limit. Large items should be split or stored in S3 with a DynamoDB pointer.

## Guardrail 8: LSIs Cannot Be Added After Table Creation
Local Secondary Indexes must be defined at table creation time. They cannot be added, modified, or removed later. GSIs can be added anytime.

## Guardrail 9: Streams Records Expire in 24 Hours
DynamoDB Streams retains records for 24 hours. If a consumer falls behind by more than 24 hours, records are permanently lost. Monitor iterator age.

## Guardrail 10: Global Tables Use Last-Writer-Wins
Concurrent writes to the same item in different regions resolve using last-writer-wins based on timestamp. There is no conflict detection or merge. Design for single-region writes per item when possible.

## Guardrail 11: Strongly Consistent Reads Bypass DAX
DAX only caches eventually consistent reads. Strongly consistent reads go directly to DynamoDB, bypassing DAX entirely. Don't expect DAX to help with strongly consistent workloads.

## Guardrail 12: Query Returns Max 1 MB Per Call
A single Query or Scan call returns at most 1 MB of data. Use LastEvaluatedKey for pagination. The 1 MB limit applies before filter expressions.
