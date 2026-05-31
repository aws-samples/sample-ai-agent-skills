---
title: "Z1 — General Outposts Troubleshooting (Catch-All)"
description: "Fallback SOP for Outposts issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Outposts issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Get Outpost overview: `aws outposts get-outpost --outpost-id <outpost-id>`
- Check connectivity: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check capacity: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name InstanceFamilyCapacityAvailability --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- List assets: `aws outposts list-assets --outpost-identifier <outpost-id>`
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=outposts.amazonaws.com --max-results 10`

SHOULD:
- Check EC2 instances on Outpost: `aws ec2 describe-instances --filters "Name=outpost-arn,Values=<outpost-arn>"`
- Check local gateway: `aws ec2 describe-local-gateways`
- Check EBS volumes: `aws ec2 describe-volumes --filters "Name=outpost-arn,Values=<outpost-arn>"`
- Check AWS Health events: `aws health describe-events --filter "services=OUTPOSTS"`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Rack connectivity / hardware → Rack (A1-A3)
- EC2 or ECS issues → Compute (B1-B2)
- EBS or S3 issues → Storage (C1-C2)
- Local gateway or service link → Networking (D1-D2)
- Updates or monitoring → Management (E1-E2)
- Region connectivity or failover → Hybrid (F1-F2)
- Encryption or IAM → Security (G1-G2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all Outpost configurations systematically
- Review CloudTrail for recent configuration changes
- Verify network connectivity end-to-end
- Check AWS Health Dashboard for service events

SHOULD:
- Compare with known-good Outpost configuration
- Check for recent on-premises network or power changes
- Review application logs on Outpost instances

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<rack|compute|storage|networking|management|hybrid|security|unknown>"
investigation_path: "outpost status → connectivity → capacity → <domain_classification>"
evidence:
  - type: outpost_config
    content: "<Outpost configuration summary>"
  - type: cloudwatch
    content: "<connectivity and capacity metrics>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — general troubleshooting uses read-only operations
- Service Disruption Risk: LOW — diagnostic commands do not affect workloads
- Compliance Impact: LOW — standard troubleshooting procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- Outpost enters an unrecoverable state
- Multiple unrelated issues occur simultaneously
- AWS Health Dashboard shows Outposts service event
- Issue requires AWS Support or on-site intervention

## Data Sensitivity
- Outpost configuration — INTERNAL
- CloudWatch metrics — INTERNAL
- Hardware health data — INTERNAL
- Error messages — INTERNAL
- Never expose Outpost site details in external communications

## Prohibited Actions
- Do NOT make changes to production Outposts without change management approval
- Do NOT run destructive commands during initial diagnosis
- Do NOT share Outpost details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt hardware repairs without AWS authorization
- Do NOT ignore recurring issues

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore Outpost to last known good configuration
- Document all troubleshooting steps and outcomes
- Verify Outpost health after troubleshooting
- Update runbooks with new findings
- Close out temporary access granted for troubleshooting

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
