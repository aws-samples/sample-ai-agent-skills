# AWS IoT Analytics Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS IoT Analytics issue.

## Guardrail 1: Channels Are Message Ingestion Points
Channels receive messages from IoT Core rules or BatchPutMessage API. Messages are stored in the channel for a configurable retention period. Channels feed into pipelines. A channel can feed multiple pipelines. Channel storage can be service-managed or customer-managed S3.

## Guardrail 2: Pipelines Process Data Sequentially
Pipeline activities execute in order. Each activity transforms the message and passes it to the next. Activity types include: channel (source), datastore (sink), lambda, addAttributes, removeAttributes, selectAttributes, filter, math, deviceRegistryEnrich, deviceShadowEnrich. Errors in one activity stop processing for that message.

## Guardrail 3: Data Stores Are NOT Databases
Data stores hold processed data from pipelines. They are optimized for batch queries via datasets, not real-time queries. Data stores can use service-managed or customer-managed S3 storage. Partitioning improves query performance for large datasets.

## Guardrail 4: Datasets Are Computed Results
SQL datasets run queries against data stores on a schedule or on-demand. Container datasets run custom Docker containers. Dataset content is the query result, not the raw data. Dataset content has a retention period. Datasets can trigger notifications when content is ready.

## Guardrail 5: SQL Dialect Is Specific to IoT Analytics
Dataset SQL supports SELECT, FROM, WHERE, GROUP BY, HAVING, ORDER BY, LIMIT. It uses Presto SQL syntax. Functions include date/time, string, math, and aggregate functions. JOINs between data stores are supported. The __dt partition column is automatically available for time-based filtering.

## Guardrail 6: Container Activities Run Docker Containers
Container datasets execute custom Docker containers on ECS. The container must read input from and write output to specific S3 paths. Container execution role must have permissions for input/output S3 buckets. Container timeout is configurable. Failed containers produce no dataset content.

## Guardrail 7: Pipeline Lambda Activities Have Limits
Lambda activities in pipelines invoke a Lambda function for each message. The function must return the transformed message. Timeout is limited. Errors cause the message to be dropped. Lambda must be in the same region as the pipeline.

## Guardrail 8: Channel Retention Affects Data Availability
Channel retention period determines how long raw messages are stored. Default is unlimited (until manually deleted). Setting retention deletes data older than the period. Channel data is separate from data store data — deleting channel data doesn't affect data stores.

## Guardrail 9: Data Store Partitioning Improves Performance
Custom partitions on data stores improve query performance. Partition keys should match common query filters. The __dt (timestamp) partition is automatic. Custom partitions are defined at data store creation and cannot be changed later.

## Guardrail 10: Batch Operations Have Size Limits
BatchPutMessage accepts up to 100 messages per call. Each message has a maximum size of 128 KB. Message IDs must be unique within a batch. Duplicate message IDs cause the entire batch to fail.

## Guardrail 11: Dataset Scheduling Has Minimum Intervals
Dataset schedules use cron expressions. Minimum interval depends on data volume and query complexity. Overlapping executions are queued. Failed executions do not automatically retry. Manual triggers are available via create-dataset-content.

## Guardrail 12: IAM Roles Are Required for Service Actions
Pipelines need IAM roles for Lambda activities and enrichment activities. Datasets need IAM roles for S3 access and container execution. Data stores with customer-managed S3 need IAM roles. Missing permissions cause silent failures.
