---
title: "F1 — Scan Frequency"
description: "Configure and troubleshoot Inspector scan frequency settings"
status: active
severity: LOW
triggers:
  - "scan frequency"
  - "how often scanned"
  - "rescan interval"
  - "continuous scanning"
owner: devops-agent
objective: "Understand and configure Inspector scanning frequency"
context: "Inspector scanning is event-driven, not scheduled. EC2 scans trigger on SSM inventory changes and new CVEs. ECR scans trigger on image push and new CVEs. ECR also supports continuous scanning configuration. Lambda scans trigger on function updates and new CVEs."
---

## Phase 1 — Triage

MUST:
- Check ECR scan frequency configuration: `aws ecr get-registry-scanning-configuration`
- Check account scanning status: `aws inspector2 batch-get-account-status`
- Verify last scan times in coverage: `aws inspector2 list-coverage --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_EC2_INSTANCE"}]}' --max-results 5`

SHOULD:
- Check if ECR continuous scanning is enabled vs scan-on-push only
- Verify SSM agent is reporting inventory updates regularly
- Check when findings were last updated

MAY:
- Review Inspector event history in CloudTrail
- Check if new CVEs have been published recently

## Phase 2 — Remediate

MUST:
- For ECR, configure continuous scanning: update registry scanning configuration with CONTINUOUS_SCAN frequency
- Ensure SSM agent is running for regular EC2 inventory updates
- Verify all scan types are enabled

SHOULD:
- Configure ECR scanning rules for appropriate frequency per repository
- Set up EventBridge rules to monitor scan activity

MAY:
- Force EC2 rescan by restarting SSM agent
- Push updated images to trigger ECR rescan

## Common Issues

- symptoms: "EC2 instance not rescanned after patching"
  diagnosis: "SSM agent hasn't reported updated inventory yet."
  resolution: "Wait for SSM inventory cycle or restart SSM agent to trigger update."

- symptoms: "ECR images only scanned on push, not continuously"
  diagnosis: "Scan frequency set to SCAN_ON_PUSH instead of CONTINUOUS_SCAN."
  resolution: "Update ECR scanning configuration to CONTINUOUS_SCAN."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Configure continuous scanning | GREEN | Enhances scanning frequency |
| Ensure SSM agent running | GREEN | Verification and restart |
| Verify scan types enabled | GREEN | Read-only verification |

## Escalation Conditions

- EC2 instances not being rescanned after patching — escalate to operations team
- ECR continuous scanning not enabled for compliance — escalate to compliance team
- Scan frequency concerns for security-critical resources — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws ecr get-registry-scanning-configuration` | MEDIUM | Exposes scanning frequency settings |
| `aws inspector2 list-coverage` | MEDIUM | Exposes last scan timestamps |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** reduce scan frequency for compliance-critical resources

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Changed ECR scan frequency | Revert to previous frequency setting |
| Restarted SSM agent | No rollback needed |

## Output Format

```yaml
root_cause: "scan_frequency — <specific_cause>"
evidence:
  - type: scan_config
    content: "<scanning frequency configuration>"
  - type: last_scan
    content: "<last scan timestamps>"
severity: LOW
mitigation:
  immediate: "Configure appropriate scan frequency"
  long_term: "Monitor scan activity and coverage freshness"
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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
