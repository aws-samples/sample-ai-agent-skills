---
title: "Z1 — General Verified Access Troubleshooting"
description: "Catch-all runbook for Verified Access issues"
status: active
severity: MEDIUM
triggers: ["verified access error", "verified access not working", "zero trust issue", "VA problem"]
owner: devops-agent
objective: "Systematic investigation of unclassified Verified Access issues"
context: "Use when the issue doesn't clearly match a specific runbook. Covers general diagnostic steps for Verified Access."
---

## Phase 1 — Triage

MUST:
- List instances: `aws ec2 describe-verified-access-instances --query 'VerifiedAccessInstances[*].{Id:VerifiedAccessInstanceId,TrustProviders:VerifiedAccessTrustProviders}'`
- Check endpoints: `aws ec2 describe-verified-access-endpoints --query 'VerifiedAccessEndpoints[*].{Id:VerifiedAccessEndpointId,GroupId:VerifiedAccessGroupId,Status:Status}'`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20`

SHOULD:
- Check access logs if logging is enabled
- Verify trust provider status
- Review endpoint connectivity

MAY:
- Check AWS Health Dashboard for Verified Access service issues
- Review recent configuration changes

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1-F1)

SHOULD:
- Document findings and investigation path

MAY:
- Enable logging if not already configured

## Safety Ratings

```
safety_ratings:
  - "describe-verified-access-instances: GREEN — read-only enumeration"
  - "describe-verified-access-endpoints: GREEN — read-only endpoint check"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Verified Access service appears non-functional in the Region
- All endpoints showing error status simultaneously
- Trust providers all returning errors
- Issue cannot be classified into any specific runbook after investigation

## Data Sensitivity

- `aws ec2 describe-verified-access-instances`: LOW — returns instance metadata
- `aws ec2 describe-verified-access-endpoints`: MEDIUM — reveals application topology
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER delete Verified Access resources during investigation
- NEVER dismiss the issue without documenting the investigation path
- NEVER modify Cedar policies as a first troubleshooting step

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore trust provider settings if modified
3. Verify all endpoints are functioning after investigation
4. Document all investigation steps and findings
5. Verify Verified Access is functioning normally before closing

## Output Format

```yaml
root_cause: "general_verified_access — <specific_cause>"
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
