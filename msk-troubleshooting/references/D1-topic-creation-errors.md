---
title: "D1 — MSK Topic Creation Errors"
description: "Diagnose and resolve topic creation failures in MSK"
status: active
severity: MEDIUM
triggers:
  - "topic creation failed"
  - "cannot create topic"
  - "TopicExistsException"
  - "InvalidReplicationFactorException"
  - "auto.create.topics"
  - "topic already exists"
owner: devops-agent
objective: "Resolve topic creation errors and establish correct topic configuration"
context: "Topic creation can fail due to invalid replication factor (exceeding broker count), insufficient permissions, topic name conflicts, broker configuration restrictions, or cluster being in a non-ACTIVE state. auto.create.topics.enable controls whether topics are created automatically on first produce/consume. MSK Serverless has a 50-topic limit with max 200 partitions per topic."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.State'`
- Check broker count: `aws kafka list-nodes --cluster-arn <cluster-arn>`
- Check broker configuration: `aws kafka describe-configuration --arn <config-arn>`
- Try to describe the topic: `kafka-topics.sh --bootstrap-server <broker>:9092 --describe --topic <topic>`

SHOULD:
- Check auto.create.topics.enable setting in broker config
- Check default.replication.factor and num.partitions in broker config
- Verify client has topic creation permissions (IAM or ACLs)
- Check if topic name contains invalid characters

MAY:
- List all existing topics: `kafka-topics.sh --bootstrap-server <broker>:9092 --list`
- Check topic count against limits (especially for Serverless)
- Review CloudTrail for topic creation API calls

## Phase 2 — Remediate

MUST:
- If replication factor > broker count: reduce replication factor to <= number of brokers
- If topic already exists: use existing topic or delete and recreate with new config
- If permissions denied: add kafka-cluster:CreateTopic IAM permission or Kafka ACL
- If cluster not ACTIVE: wait for cluster to become ACTIVE before creating topics

SHOULD:
- Set appropriate partition count based on throughput needs (cannot decrease later)
- Set replication factor = 3 for production topics
- Configure retention.ms and retention.bytes at topic creation
- Disable auto.create.topics.enable in production to prevent accidental topic creation

MAY:
- Use topic naming conventions for organization
- Set topic-level configs at creation time (cleanup.policy, compression.type, etc.)
- Create topics via infrastructure as code (Terraform, CloudFormation)

## Common Issues

- symptoms: "InvalidReplicationFactorException when creating topic"
  diagnosis: "Replication factor exceeds the number of available brokers."
  resolution: "Set replication factor <= broker count. For a 3-broker cluster, max replication factor is 3."

- symptoms: "Topics appearing that nobody created"
  diagnosis: "auto.create.topics.enable is true (default). Topics created on first produce/consume."
  resolution: "Set auto.create.topics.enable=false in broker configuration. Delete unwanted topics."

- symptoms: "Topic creation fails with authorization error"
  diagnosis: "Client IAM role missing kafka-cluster:CreateTopic permission."
  resolution: "Add kafka-cluster:CreateTopic to the IAM policy with the topic ARN as resource."

## Output Format

```yaml
root_cause: "topic_creation_error — <specific_cause>"
evidence:
  - type: cluster_config
    content: "<broker count, configuration>"
  - type: topic_details
    content: "<topic describe output or error message>"
  - type: permissions
    content: "<IAM policy or ACL configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix topic configuration or permissions and retry"
  long_term: "Standardize topic creation process, disable auto-create in production"
```


## Safety Ratings
```
safety_ratings:
  - "Check cluster status: GREEN — read-only API call"
  - "Describe topic: GREEN — read-only Kafka CLI command"
  - "List existing topics: GREEN — read-only Kafka CLI command"
  - "Create topic with correct configuration: GREEN — adds new resource, no impact on existing topics"
  - "Delete and recreate topic: RED — permanently removes topic and all messages"
  - "Disable auto.create.topics.enable: YELLOW — broker config change triggers rolling restart"
```

## Escalation Conditions
- Cluster serves production event streaming
- Topic creation failure blocking application deployment
- Fix requires broker configuration change (triggers rolling restart)
- Topic already exists with wrong configuration and has active producers/consumers
- Serverless topic limit reached requiring architecture review

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Topic names: application and data flow identifiers"
    - "Topic configurations: retention, replication settings"
    - "IAM policies: access control for topic operations"
  handling: "Do not expose topic naming conventions or configurations externally."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER delete a topic to recreate it without confirming all consumers have been stopped and data loss is acceptable
- NEVER enable auto.create.topics.enable in production environments
- NEVER create topics with replication factor 1 in production

## Phase 3 — Rollback
- If a topic was created with wrong configuration: delete the topic (if no active consumers) and recreate with correct settings
- If auto.create.topics.enable was changed: revert broker configuration and apply (triggers rolling restart)
- If default.replication.factor was changed: revert broker configuration to previous value
- If IAM permissions were added: remove the added policy statements to restore previous access
- If unwanted auto-created topics exist: delete them after confirming no active consumers

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing authentication"
  - "NEVER suggest public access to brokers"
