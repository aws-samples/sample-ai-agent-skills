---
name: kinesis-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Kinesis Data
  Streams and Amazon Kinesis Data Firehose problems by analyzing stream
  configurations, shard throughput, consumer lag, producer failures,
  Firehose delivery issues, Lambda transformations, capacity management,
  data integrity, and security. Activate when: write/read throttling,
  hot shards, resharding failures, iterator age lag, KCL checkpoint
  issues, enhanced fan-out problems, Lambda consumer errors, PutRecords
  partial failures, KPL aggregation issues, Firehose S3/OpenSearch/
  Redshift/HTTP delivery failures, Lambda transformation errors, data
  format conversion problems, on-demand scaling issues, provisioned
  shard management, data loss investigation, ordering guarantees,
  encryption configuration, cross-account access, or the user says
  something is wrong with Kinesis without naming specific symptoms.
compatibility: >
  Requires AWS CLI with kinesis, firehose, cloudwatch, and cloudtrail
  permissions. Optional: KCL application logs, KPL logs, Lambda logs,
  kubectl (for EKS-based consumers/producers).
---

# Kinesis Diagnostics

## When to use

Any Kinesis investigation where the console alone is insufficient — Data Streams throughput issues, consumer lag, producer failures, Firehose delivery problems, Lambda transformation errors, capacity management, data integrity concerns, or security configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
# Describe the Kinesis Data Stream
aws kinesis describe-stream-summary --stream-name <stream-name>

# List shards and their status
aws kinesis list-shards --stream-name <stream-name>

# Get shard iterator for reading records
aws kinesis get-shard-iterator --stream-name <stream-name> \
  --shard-id <shard-id> --shard-iterator-type LATEST

# Describe Firehose delivery stream
aws firehose describe-delivery-stream --delivery-stream-name <stream-name>

# Check CloudWatch metrics for Data Streams
# Key metrics: IncomingBytes, IncomingRecords, WriteProvisionedThroughputExceeded,
# ReadProvisionedThroughputExceeded, GetRecords.IteratorAgeMilliseconds
aws cloudwatch get-metric-statistics --namespace AWS/Kinesis \
  --metric-name WriteProvisionedThroughputExceeded \
  --dimensions Name=StreamName,Value=<stream-name> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Sum

# Check Firehose CloudWatch metrics
# Key metrics: DeliveryToS3.Success, DeliveryToS3.Records, DataFreshness
aws cloudwatch get-metric-statistics --namespace AWS/Firehose \
  --metric-name DeliveryToS3.Success \
  --dimensions Name=DeliveryStreamName,Value=<stream-name> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Average
```

Triage returns:
- Stream status, shard count, retention period, capacity mode
- Write and read throughput exceeded counts
- Iterator age (consumer lag) per shard
- Firehose delivery success rate and data freshness
- Firehose destination configuration and error logs

If the stream status is not ACTIVE, that IS the root cause domain. Don't chase throughput or consumer symptoms.

### Step 2 — Domain deep dive (only if needed)

```
# Check per-shard metrics for hot shard detection
aws cloudwatch get-metric-statistics --namespace AWS/Kinesis \
  --metric-name IncomingBytes \
  --dimensions Name=StreamName,Value=<stream-name> Name=ShardId,Value=<shard-id> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Sum

# Check iterator age for consumer lag
aws cloudwatch get-metric-statistics --namespace AWS/Kinesis \
  --metric-name GetRecords.IteratorAgeMilliseconds \
  --dimensions Name=StreamName,Value=<stream-name> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Maximum

# List registered consumers (enhanced fan-out)
aws kinesis list-stream-consumers --stream-arn <stream-arn>

# Describe consumer for enhanced fan-out status
aws kinesis describe-stream-consumer --stream-arn <stream-arn> \
  --consumer-name <consumer-name>

# Check Firehose error logging in CloudWatch Logs
aws logs filter-log-events --log-group-name /aws/kinesisfirehose/<stream-name> \
  --start-time $(date -u -d '1 hour ago' +%s)000 --limit 20

# CloudTrail for Kinesis API events
aws cloudtrail lookup-events --lookup-attributes \
  AttributeKey=EventSource,AttributeValue=kinesis.amazonaws.com --max-results 20
```

Read `references/kinesis-guardrails.md` before concluding on any Kinesis issue.

### Step 3 — Detailed investigation (low-confidence cases only)

```
# CloudTrail for Firehose API events
aws cloudtrail lookup-events --lookup-attributes \
  AttributeKey=EventSource,AttributeValue=firehose.amazonaws.com --max-results 20

# Check DynamoDB table for KCL checkpoint data
aws dynamodb scan --table-name <kcl-app-name> --max-items 10

# Check Lambda function for Kinesis event source mapping
aws lambda list-event-source-mappings --function-name <function-name> \
  --event-source-arn <stream-arn>

# Check enhanced monitoring (shard-level metrics)
aws kinesis describe-stream --stream-name <stream-name> \
  --query 'StreamDescription.EnhancedMonitoring'

# Verify encryption configuration
aws kinesis describe-stream --stream-name <stream-name> \
  --query 'StreamDescription.EncryptionType'
```

## Tool quick reference

| Tool / Command | When to use |
|----------------|-------------|
| `aws kinesis describe-stream-summary` | Stream status, shard count, capacity mode, retention |
| `aws kinesis list-shards` | List all shards, check for OPEN/CLOSED status |
| `aws kinesis get-shard-iterator` | Get iterator to read records from a shard |
| `aws kinesis get-records` | Read records from a shard for data verification |
| `aws kinesis list-stream-consumers` | List enhanced fan-out consumers |
| `aws kinesis describe-stream-consumer` | Consumer registration status |
| `aws firehose describe-delivery-stream` | Firehose config, destination, buffering, transforms |
| `aws cloudwatch get-metric-statistics` | Throughput, throttling, iterator age, Firehose metrics |
| `aws cloudtrail lookup-events` | Kinesis/Firehose API call history and errors |
| `aws dynamodb scan` | KCL checkpoint table inspection |
| `aws lambda list-event-source-mappings` | Lambda consumer event source mapping config |
| `aws logs filter-log-events` | Firehose error logs in CloudWatch Logs |

## Gotchas: Kinesis

These are the mistakes commonly made during Kinesis troubleshooting.

- Each shard supports 1 MB/s or 1,000 records/s for writes and 2 MB/s for reads. These are hard limits per shard — you cannot increase them. If you need more throughput, add more shards.
- Hot shards occur when partition keys are poorly distributed. If most records hash to the same shard, that shard hits its throughput limit while others sit idle. Use random or well-distributed partition keys.
- IteratorAgeMilliseconds is the key metric for consumer lag. It measures how far behind the consumer is from the tip of the stream. High iterator age means the consumer cannot keep up with the producer write rate.
- Enhanced fan-out provides 2 MB/s per shard per consumer via SubscribeToShard (push model). Shared throughput (GetRecords) shares the 2 MB/s read limit across ALL consumers on a shard. Use enhanced fan-out when you have multiple consumers or need low latency.
- KCL uses a DynamoDB table for checkpoint management. If the DynamoDB table has insufficient capacity or is throttled, KCL workers cannot checkpoint, causing duplicate processing on restart. Check the DynamoDB table's consumed capacity.
- Firehose buffers data before delivery. Buffer size (1-128 MB) and buffer interval (60-900 seconds) control delivery frequency. Whichever threshold is reached first triggers delivery. Smaller buffers = more frequent delivery but more small files.
- Firehose Lambda transformations must return records in the exact expected format with recordId, result (Ok/Dropped/ProcessingFailed), and base64-encoded data. Malformed responses cause the entire batch to fail.
- Firehose delivery failures to S3, OpenSearch, or Redshift are logged to CloudWatch Logs if error logging is enabled. Always check `/aws/kinesisfirehose/<stream-name>` for delivery error details.
- Data retention for Kinesis Data Streams is configurable from 24 hours (default) to 365 days (8,760 hours). Extended retention increases cost. Data beyond the retention period is permanently lost.
- Resharding (split/merge) changes the shard structure. During resharding, parent shards close and child shards open. Consumers must handle the transition — KCL handles this automatically, but custom consumers must track shard lineage.
- PutRecords API can return partial failures — the overall API call succeeds (HTTP 200) but individual records may fail. Always check the `FailedRecordCount` in the response and retry failed records with exponential backoff.
- KPL aggregates multiple user records into a single Kinesis record to improve throughput. Consumers must use KCL or the KPL deaggregation library to unpack aggregated records. Raw GetRecords calls return aggregated blobs.
- Deregistering an enhanced fan-out consumer does not happen instantly. The consumer enters DELETING state and may take a few minutes. You cannot re-register a consumer with the same name until deletion completes.
- Firehose dynamic partitioning routes records to different S3 prefixes based on record content (JQ expressions or Lambda). Misconfigured partitioning keys cause all records to land in a single prefix or fail delivery.
- On-demand capacity mode automatically scales shards based on traffic. However, it can take several minutes to scale up, and there is a maximum throughput that scales to 200 MB/s write and 400 MB/s read. Provisioned mode gives you explicit control over shard count.

## Anti-hallucination rules

1. Always cite specific AWS CLI output, CloudWatch metrics, CloudTrail events, or application logs as evidence.
2. Never suggest increasing per-shard throughput limits — they are fixed at 1 MB/s write and 2 MB/s read per shard. The only way to increase total throughput is to add more shards.
3. Never claim Firehose delivers data in real-time — it buffers data and delivers in batches based on buffer size and interval settings.
4. Never suggest accessing Kinesis internal infrastructure — it is fully managed. You interact via APIs only.
5. Never assume KPL-aggregated records can be read directly without deaggregation — consumers must use KCL or the deaggregation library.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Data Streams Throughput | A1–A4 | Write throttling, read throttling, hot shards, resharding issues |
| B — Consumers | B1–B4 | Iterator age lag, KCL issues, enhanced fan-out, Lambda consumer |
| C — Producers | C1–C3 | PutRecords failures, KPL issues, aggregation problems |
| D — Firehose Delivery | D1–D4 | S3 delivery failures, OpenSearch delivery, Redshift delivery, HTTP endpoint |
| E — Firehose Transform | E1–E2 | Lambda transformation errors, data format conversion |
| F — Capacity | F1–F2 | On-demand scaling, provisioned shard management |
| G — Data Integrity | G1–G2 | Data loss investigation, ordering guarantees |
| H — Security | H1–H2 | Encryption, cross-account access |
| Z — Catch-All | Z1 | General Kinesis troubleshooting |
