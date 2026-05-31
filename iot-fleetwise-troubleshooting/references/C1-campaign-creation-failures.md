---
title: "C1 — Campaign Creation Failures"
description: "Diagnose campaign creation, approval, and configuration failures"
status: active
severity: HIGH
triggers: ["campaign creation failed", "campaign not starting", "campaign validation error", "cannot approve campaign"]
owner: devops-agent
objective: "Resolve campaign creation and activation issues to enable data collection"
context: "Campaigns define data collection rules and target fleets. They require approval before starting. Campaigns reference signals from model manifests and send data to S3 or Timestream destinations."
---

## Phase 1 — Triage

MUST:
- List campaigns: `aws iotfleetwise list-campaigns --query 'campaignSummaries[*].{Name:name,Status:status,TargetArn:targetArn}'`
- Describe the failing campaign: `aws iotfleetwise get-campaign --name <name>`
- Check campaign status and target fleet: verify status is not stuck in CREATING or FAILED

SHOULD:
- Verify target fleet exists and has vehicles: `aws iotfleetwise list-vehicles-in-fleet --fleet-id <fleet-id>`
- Check campaign signal references match model manifest signals
- Verify destination (S3 bucket or Timestream table) is accessible

MAY:
- Review CloudTrail for CreateCampaign events
- Check campaign collection scheme configuration (time-based or condition-based)
- Verify campaign data destination IAM permissions

## Phase 2 — Remediate

MUST:
- Ensure target fleet exists and contains at least one vehicle
- Fix signal references to match signals defined in vehicle model manifests
- Configure valid destination (S3 or Timestream) with correct permissions
- Approve campaign after creation: `aws iotfleetwise update-campaign --name <name> --action APPROVE`

SHOULD:
- Validate collection scheme parameters (interval, condition expression)
- Ensure campaign compression and data format settings are correct
- Verify IAM role for destination has write permissions

MAY:
- Test campaign with a small fleet before production deployment
- Set up campaign monitoring via CloudWatch metrics
- Configure campaign expiry time if data collection is time-bounded

## Safety Ratings

```
safety_ratings:
  - "list-campaigns: GREEN — read-only enumeration"
  - "get-campaign: GREEN — read-only inspection"
  - "create-campaign: YELLOW — creates campaign resource, does not start collection"
  - "update-campaign --action APPROVE: YELLOW — starts data collection on target fleet"
  - "update-campaign --action SUSPEND: YELLOW — pauses data collection"
  - "delete-campaign: RED — permanently removes campaign and stops collection"
```

## Escalation Conditions

- Campaign creation fails with internal errors after fixing all configuration
- Campaign approved but no data appears in destination after 30 minutes
- Campaign targets wrong fleet and is already collecting data
- Destination (S3/Timestream) access issues require cross-account permission changes

## Data Sensitivity

- `aws iotfleetwise get-campaign`: MEDIUM — contains collection rules and destination configuration
- `aws iotfleetwise list-vehicles-in-fleet`: MEDIUM — returns vehicle identifiers
- Campaign data in S3/Timestream: HIGH — contains vehicle telemetry data

## Prohibited Actions

- NEVER approve a campaign targeting production fleet without testing on staging fleet first
- NEVER delete a campaign to stop collection — suspend it first to allow resumption
- NEVER create campaigns with overly broad collection schemes that generate excessive data and cost
- NEVER configure campaign destinations without verifying IAM permissions first

## Phase 3 — Rollback

1. Suspend the campaign: `aws iotfleetwise update-campaign --name <name> --action SUSPEND`
2. If campaign was created with wrong configuration, delete and recreate: `aws iotfleetwise delete-campaign --name <name>`
3. If wrong fleet was targeted, create new campaign with correct fleet ARN
4. Verify no unwanted data was collected in the destination
5. Clean up any test data from S3 or Timestream if needed

## Common Issues

- symptoms: "CreateCampaign returns ValidationException about signal references"
  diagnosis: "Campaign references signals not defined in the model manifests of vehicles in the target fleet."
  resolution: "Verify signal fully qualified names match model manifest signal paths exactly."

- symptoms: "Campaign status is WAITING_FOR_APPROVAL indefinitely"
  diagnosis: "Campaign requires explicit approval to start. It does not auto-approve."
  resolution: "Approve the campaign: aws iotfleetwise update-campaign --name <name> --action APPROVE"

## Output Format

```yaml
root_cause: "campaign_creation — <specific_cause>"
evidence:
  - type: campaign_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix campaign configuration and approve"
  long_term: "Implement campaign validation and staged rollout process"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
