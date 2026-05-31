---
title: "A2 — PCI DSS Compliance Issues"
description: "Diagnose PCI DSS standard control failures in Security Hub"
status: active
severity: HIGH
triggers:
  - "PCI DSS"
  - "PCI compliance"
  - "PCI control failed"
  - "payment card"
  - "PCI standard"
owner: devops-agent
objective: "Identify and resolve PCI DSS standard control failures in Security Hub"
context: "The PCI DSS standard in Security Hub evaluates controls related to payment card data security. Controls cover network security, access control, encryption, logging, and vulnerability management. PCI DSS compliance in Security Hub is a subset of the full PCI DSS assessment."
---

## Phase 1 — Triage

MUST:
- Check if PCI DSS standard is enabled: `aws securityhub get-enabled-standards`
- List PCI DSS controls: `aws securityhub describe-standards-controls --standards-subscription-arn <pci-standard-arn>`
- Get failed PCI findings: `aws securityhub get-findings --filters '{"ComplianceStatus":[{"Value":"FAILED","Comparison":"EQUALS"}],"GeneratorId":[{"Value":"pci-dss","Comparison":"PREFIX"}]}' --max-results 20`
- Verify Config is enabled: `aws configservice describe-configuration-recorder-status`
- Check the specific resources failing PCI controls

SHOULD:
- Verify encryption settings for resources handling cardholder data
- Check network segmentation and security group configurations
- Review IAM access controls for PCI-scoped resources

MAY:
- Compare Security Hub PCI controls against full PCI DSS requirements
- Check for PCI-specific Config rules

## Phase 2 — Remediate

MUST:
- Fix the specific resource configuration causing the control failure
- Ensure encryption is enabled for data at rest and in transit
- Verify network security controls are properly configured

SHOULD:
- Enable all PCI DSS controls for comprehensive coverage
- Document remediation actions for PCI audit evidence
- Set up automated remediation for common PCI failures

MAY:
- Create a PCI-specific compliance dashboard
- Set up notifications for PCI control status changes

## Common Issues

- symptoms: "PCI DSS controls show NO_DATA for encryption checks"
  diagnosis: "Config is not recording the resource types needed for encryption controls."
  resolution: "Enable Config with all supported resource types. Ensure KMS, S3, RDS, and EBS resource types are recorded."

- symptoms: "PCI network security controls failing"
  diagnosis: "Security groups allow unrestricted access or VPC configurations do not meet PCI segmentation requirements."
  resolution: "Restrict security group rules, implement network segmentation, and use NACLs for additional control."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix resource configuration | YELLOW | Resource change; varies by resource |
| Enable encryption | YELLOW | Configuration change; reversible |
| Fix network security controls | YELLOW | Security group change; reversible |

## Escalation Conditions

- Standard enablement/disablement
- Cross-account configuration changes
- PCI-scoped resource changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-enabled-standards` | LOW | Standard subscription status |
| `describe-standards-controls` | LOW | Control status |
| `get-findings` (PCI) | MEDIUM | PCI compliance findings |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Resource configuration fix | Revert resource to previous configuration |
| Encryption enablement | Cannot disable encryption on most resources; plan carefully |
| Security group change | Revert security group rules |

## Output Format

```yaml
root_cause: "pci_dss_failure — <specific_control>"
evidence:
  - type: standard_controls
    content: "<PCI DSS control status>"
  - type: findings
    content: "<failed PCI findings>"
severity: HIGH
mitigation:
  immediate: "Fix the specific PCI control failure"
  long_term: "Implement continuous PCI compliance monitoring"
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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
