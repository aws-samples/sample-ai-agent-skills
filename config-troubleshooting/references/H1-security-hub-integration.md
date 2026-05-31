---
title: "H1 — Security Hub Integration"
description: "Diagnose AWS Config integration issues with AWS Security Hub"
status: active
severity: MEDIUM
triggers:
  - "security hub config"
  - "config security hub"
  - "security hub controls"
  - "config rules security hub"
  - "security hub integration"
owner: devops-agent
objective: "Identify and resolve integration issues between AWS Config and Security Hub"
context: "Security Hub relies on AWS Config for most of its security controls. Config rules power Security Hub controls, and Config must be enabled with the correct resource types for controls to evaluate. Integration issues manifest as controls showing no data or incorrect compliance status."
---

## Phase 1 — Triage

MUST:
- Verify Config is enabled: `aws configservice describe-configuration-recorder-status`
- Check Security Hub is enabled: `aws securityhub describe-hub`
- List enabled standards: `aws securityhub get-enabled-standards`
- Check control status: `aws securityhub describe-standards-controls --standards-subscription-arn <standard-arn>`
- Verify Config records the resource types needed by Security Hub controls

SHOULD:
- Check for Config rules created by Security Hub: `aws configservice describe-config-rules` (look for securityhub prefix)
- Verify the Config service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForConfig`
- Check if Security Hub findings are being generated: `aws securityhub get-findings --filters '{"ProductName":[{"Value":"Security Hub","Comparison":"EQUALS"}]}' --max-results 10`

MAY:
- Check CloudTrail for Security Hub and Config interaction events
- Verify cross-region aggregation if using Security Hub aggregation

## Phase 2 — Remediate

MUST:
- Enable Config recorder if not running
- Ensure Config records all resource types required by Security Hub controls
- Enable the required security standards in Security Hub

SHOULD:
- Enable "Record all supported resource types" for comprehensive Security Hub coverage
- Enable global resource recording for IAM-related controls
- Re-enable controls that were disabled

MAY:
- Set up cross-region aggregation for centralized Security Hub findings
- Configure Security Hub custom actions for automated remediation

## Common Issues

- symptoms: "Security Hub controls show 'No data available' or UNKNOWN status"
  diagnosis: "Config is not enabled or not recording the resource types required by those controls."
  resolution: "Enable Config recorder with all supported resource types. Wait for initial evaluation (up to 24 hours)."

- symptoms: "Security Hub controls show stale compliance data"
  diagnosis: "Config recorder was stopped and restarted, or Config rules are not evaluating."
  resolution: "Verify Config recorder is running and rules are evaluating. Force re-evaluation if needed."

- symptoms: "Config rules exist but Security Hub does not show corresponding controls"
  diagnosis: "The security standard containing those controls is not enabled in Security Hub."
  resolution: "Enable the appropriate security standard (CIS, PCI DSS, AWS Foundational) in Security Hub."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Config recorder | YELLOW | Service enablement; reversible |
| Enable all resource types | YELLOW | Configuration change; reversible |
| Enable security standards | YELLOW | Standard enablement; reversible |

## Escalation Conditions

- Recorder configuration changes
- Security standard enablement/disablement

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-configuration-recorder-status` | LOW | Recorder status |
| `describe-hub` | LOW | Security Hub status |
| `get-findings` (Security Hub) | HIGH | Security findings |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Config recorder enablement | Stop recorder (not recommended) |
| Resource type change | Revert recorder configuration |
| Standard enablement | Disable standard |

## Output Format

```yaml
root_cause: "security_hub_integration — <specific_cause>"
evidence:
  - type: config_status
    content: "<Config recorder and rule status>"
  - type: security_hub
    content: "<Security Hub standards and control status>"
  - type: findings
    content: "<relevant Security Hub findings>"
severity: MEDIUM
mitigation:
  immediate: "Enable Config with required resource types for Security Hub controls"
  long_term: "Monitor both Config and Security Hub status for integration health"
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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
