---
title: "A1 — Index Creation Failures"
description: "Diagnose Resource Explorer index creation failures"
status: active
severity: HIGH
triggers: ["cannot create index", "index creation failed", "CreateIndex error"]
owner: devops-agent
objective: "Resolve index creation issues for Resource Explorer"
context: "Each region needs a local index. Only one aggregator index per account. Index creation is asynchronous."
---

## Phase 1 — Triage

MUST:
- Check existing indexes: `aws resource-explorer-2 list-indexes --query 'Indexes[*].{Region:Region,Type:Type,Arn:Arn}'`
- Verify IAM permissions for resource-explorer-2:CreateIndex: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names resource-explorer-2:CreateIndex`
- Check if an index already exists in the region: `aws resource-explorer-2 get-index`

SHOULD:
- Verify no SCP blocks Resource Explorer actions
- Check CloudTrail for CreateIndex errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateIndex --max-results 10`

MAY:
- Review AWS service quotas for Resource Explorer
- Check if the region supports Resource Explorer

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for index creation
- If aggregator already exists, create a LOCAL index instead
- Wait for index status to transition to ACTIVE

SHOULD:
- Create indexes in all regions with resources
- Use CloudFormation or Terraform for consistent multi-region deployment

MAY:
- Set up CloudWatch alarms for index status changes
- Document the index topology (which region has the aggregator)

## Safety Ratings

```
safety_ratings:
  - "resource-explorer-2 list-indexes: GREEN — read-only index enumeration"
  - "resource-explorer-2 get-index: GREEN — read-only index details"
  - "iam simulate-principal-policy: GREEN — read-only permission check"
  - "resource-explorer-2 create-index: YELLOW — creates index resource"
```

## Escalation Conditions

- Index creation fails repeatedly with internal service errors
- SCP blocks Resource Explorer and requires organization admin changes
- Index stuck in CREATING state for more than 1 hour
- Multiple regions need simultaneous index creation in production

## Data Sensitivity

- `aws resource-explorer-2 list-indexes`: LOW — returns index metadata
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principals

## Prohibited Actions

- NEVER delete an existing aggregator index to create a new one without planning for re-indexing delay
- NEVER create an aggregator index without understanding it replaces cross-region search topology
- NEVER grant resource-explorer-2:* to troubleshoot creation — use least privilege
- NEVER assume index creation failure means the service is unavailable

## Phase 3 — Rollback

1. If an index was created in the wrong region, delete it: `aws resource-explorer-2 delete-index --arn <index-arn> --type <type>`
2. If an aggregator was created in the wrong region, plan migration with 36-hour re-indexing window
3. Verify remaining indexes are ACTIVE: `aws resource-explorer-2 list-indexes`
4. Restore IAM policies if they were modified during troubleshooting
5. Document the index topology for the team

## Common Issues

- symptoms: "CreateIndex returns ConflictException"
  diagnosis: "An index already exists in this region, or an aggregator already exists in the account."
  resolution: "Check existing indexes with list-indexes. Only one aggregator per account. Use LOCAL type for additional regions."

## Output Format

```yaml
root_cause: "index_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Deploy indexes via IaC"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
