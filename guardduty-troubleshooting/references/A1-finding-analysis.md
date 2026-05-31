---
title: "A1 — Finding Analysis"
description: "Analyze and investigate Amazon GuardDuty findings"
status: active
severity: HIGH
triggers:
  - "guardduty finding"
  - "finding investigation"
  - "threat detected"
  - "suspicious activity"
  - "finding details"
owner: devops-agent
objective: "Systematically analyze GuardDuty findings to determine threat severity, scope, and appropriate response"
context: "GuardDuty findings indicate potential security threats detected through VPC Flow Logs, DNS logs, CloudTrail, and other data sources. Proper analysis requires understanding the finding type, affected resources, actor details, and contextual evidence."
---

## Phase 1 — Triage

MUST:
- List the detector: `aws guardduty list-detectors`
- List high-severity findings: `aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"severity":{"Gte":7}}}' --max-results 20`
- Get finding details: `aws guardduty get-findings --detector-id <detector-id> --finding-ids <finding-id>`
- Check finding statistics: `aws guardduty get-findings-statistics --detector-id <detector-id> --finding-statistic-types COUNT_BY_SEVERITY`
- Identify the affected resource (EC2 instance, IAM principal, S3 bucket, etc.)

SHOULD:
- Check the finding type prefix to understand the data source (Recon, UnauthorizedAccess, Trojan, etc.)
- Verify the actor details (remote IP, user agent, API caller)
- Check for related findings on the same resource: `aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"resource.instanceDetails.instanceId":{"Eq":["<instance-id>"]}}}'`
- Correlate with CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=<resource-id> --max-results 20`

MAY:
- Check the finding's action details (network connection, DNS request, API call)
- Verify the remote IP reputation using external threat intelligence
- Check for findings across multiple resources indicating a broader attack

## Phase 2 — Remediate

MUST:
- For HIGH severity findings, initiate incident response immediately
- Isolate compromised resources if active compromise is confirmed
- Archive findings after investigation: `aws guardduty archive-findings --detector-id <detector-id> --finding-ids <finding-id>`

SHOULD:
- Document the investigation findings and response actions
- Update security groups or NACLs to block malicious IPs
- Rotate compromised credentials if IAM-related findings

MAY:
- Create suppression rules for confirmed false positives
- Set up automated response via EventBridge for recurring finding types

## Common Issues

- symptoms: "High-severity finding for EC2 instance communicating with known threat IP"
  diagnosis: "The instance may be compromised or running software that contacts known malicious infrastructure."
  resolution: "Isolate the instance, capture forensic evidence (EBS snapshot), investigate running processes, and remediate."

- symptoms: "UnauthorizedAccess finding for IAM credentials used from unusual location"
  diagnosis: "IAM credentials may be compromised and used from an unauthorized location."
  resolution: "Disable the access keys, revoke active sessions, rotate credentials, and review CloudTrail for unauthorized actions."

- symptoms: "Recon finding showing port scanning activity"
  diagnosis: "An EC2 instance is performing port scanning, possibly due to compromise or misconfigured software."
  resolution: "Investigate the instance for malware, check security groups, and block outbound scanning."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Isolate compromised resources | RED | Destructive to availability; may not be reversible |
| Archive findings | YELLOW | State change; findings can be unarchived |
| Update security groups/NACLs | YELLOW | Network change; reversible |
| Rotate credentials | RED | Invalidates existing sessions; irreversible |

## Escalation Conditions

- HIGH severity findings requiring incident response
- Active compromise confirmed on production resources
- Suppression rule creation
- Detector configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-findings` | HIGH | Threat details, resource IDs, actor IPs |
| `list-findings` | MEDIUM | Finding IDs and metadata |
| `get-findings-statistics` | LOW | Aggregate severity counts |
| CloudTrail events | HIGH | API call details with principals |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection
- NEVER archive HIGH severity findings without investigation

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Resource isolation | Restore original security group rules |
| Finding archival | Unarchive via `unarchive-findings` |
| Security group change | Revert security group rules |
| Credential rotation | Cannot undo; issue new credentials if needed |

## Output Format

```yaml
root_cause: "finding_analysis — <finding_type>"
evidence:
  - type: finding
    content: "<get-findings output with key details>"
  - type: resource
    content: "<affected resource details>"
  - type: actor
    content: "<actor/threat details>"
severity: HIGH
mitigation:
  immediate: "Isolate affected resources and block threat actors"
  long_term: "Implement automated response and improve detection coverage"
```

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
  - command: "get-findings"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-detectors"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-threat-intel-set"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling GuardDuty to stop alerts"
  - "NEVER suggest archiving findings without investigation"
  - "NEVER expose finding details in public logs"
