---
title: "Z1 — General Outposts Servers Troubleshooting"
description: "Catch-all runbook for Outposts Server issues"
status: active
severity: MEDIUM
triggers: ["outpost server error", "outpost not working", "outpost issue", "edge server problem"]
owner: devops-agent
objective: "Systematic investigation of unclassified Outposts Server issues"
context: "Use when the issue doesn't clearly match a specific runbook. Covers general diagnostic steps for Outposts Servers."
---

## Phase 1 — Triage

MUST:
- Check Outpost status: `aws outposts get-outpost --outpost-id <outpost-id> --query '{Status:LifeCycleStatus,Name:Name,Id:OutpostId,SiteId:SiteId}'`
- Verify service link: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=outposts.amazonaws.com --max-results 20`

SHOULD:
- List instances on the Outpost and check their states
- Verify EBS volumes are healthy
- Check network connectivity (LGW, CoIP)

MAY:
- Check AWS Health Dashboard for Outposts service issues
- Review recent changes in the account

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1-F1)

SHOULD:
- Document findings and investigation path

MAY:
- Set up comprehensive monitoring if not already configured

## Safety Ratings

```
safety_ratings:
  - "get-outpost: GREEN — read-only status check"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Outpost Server appears non-functional (lifecycle status not ACTIVE)
- Service link down for extended period with no identified cause
- Multiple subsystems affected simultaneously
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws outposts get-outpost`: LOW — returns Outpost metadata
- `aws cloudwatch get-metric-statistics`: LOW — returns connectivity metrics
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without first identifying root cause
- NEVER reboot or power cycle the Outposts Server without AWS Support guidance
- NEVER dismiss the issue without documenting the investigation path
- NEVER assume the issue is hardware-related without exhausting software diagnostics

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore network settings if they were modified during troubleshooting
3. Verify all instances and volumes are in their pre-investigation state
4. Document all investigation steps and findings
5. Verify the Outposts Server is functioning normally before closing

## Output Format

```yaml
root_cause: "general_outposts_server — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and documentation"
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
