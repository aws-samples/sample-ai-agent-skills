---
title: "C3 — Lambda Protection Issues"
description: "Diagnose GuardDuty Lambda protection and serverless threat detection issues"
status: active
severity: MEDIUM
triggers:
  - "Lambda protection"
  - "Lambda findings"
  - "serverless threats"
  - "Lambda network activity"
  - "Lambda suspicious"
owner: devops-agent
objective: "Identify and resolve Lambda protection issues for serverless threat detection"
context: "GuardDuty Lambda Protection monitors network activity from Lambda functions to detect suspicious behavior such as communication with malicious IPs, cryptocurrency mining, or data exfiltration. It analyzes VPC and non-VPC Lambda function network activity."
---

## Phase 1 — Triage

MUST:
- Check Lambda protection status: `aws guardduty get-detector --detector-id <detector-id>` (check LAMBDA_NETWORK_LOGS feature)
- List Lambda-related findings: `aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"resource.resourceType":{"Eq":["Lambda"]}}}' --max-results 10`
- Verify Lambda functions exist: `aws lambda list-functions --max-items 20`
- Check the detector is ENABLED

SHOULD:
- Verify the service-linked role has Lambda-related permissions
- Check for Lambda functions with VPC configuration: `aws lambda list-functions` (check VpcConfig)
- Review organization configuration for Lambda protection: `aws guardduty describe-organization-configuration --detector-id <detector-id>`

MAY:
- Check Lambda function invocation patterns for correlation
- Verify Lambda functions have network activity that would trigger findings

## Phase 2 — Remediate

MUST:
- Enable Lambda protection if disabled: `aws guardduty update-detector --detector-id <detector-id> --features '[{"Name":"LAMBDA_NETWORK_LOGS","Status":"ENABLED"}]'`
- Verify the feature status changed to ENABLED

SHOULD:
- Enable Lambda protection across all member accounts
- Monitor Lambda finding generation after enabling

MAY:
- Set up automated response for Lambda-related findings
- Review Lambda function configurations for security best practices

## Common Issues

- symptoms: "No Lambda-related findings despite Lambda functions running"
  diagnosis: "Lambda protection is not enabled or Lambda functions have no suspicious network activity."
  resolution: "Enable LAMBDA_NETWORK_LOGS feature. Note that findings are only generated for suspicious activity."

- symptoms: "Lambda protection enabled but only seeing findings for VPC Lambda functions"
  diagnosis: "Both VPC and non-VPC Lambda functions are monitored. Findings depend on the type of suspicious activity detected."
  resolution: "Verify the feature is enabled. Allow time for baseline establishment."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Lambda protection | YELLOW | Configuration change; reversible |
| Enable across member accounts | YELLOW | Org-wide change; reversible |

## Escalation Conditions

- Detector configuration changes
- Lambda protection enablement/disablement

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` (Lambda feature) | LOW | Feature status |
| `list-findings` (Lambda) | HIGH | Lambda threat details |
| `list-functions` | MEDIUM | Function names and configurations |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Lambda protection enablement | Disable via `update-detector` (not recommended) |

## Output Format

```yaml
root_cause: "lambda_protection — <specific_cause>"
evidence:
  - type: detector_features
    content: "<Lambda protection status>"
  - type: findings
    content: "<Lambda-related findings>"
severity: MEDIUM
mitigation:
  immediate: "Enable Lambda protection"
  long_term: "Monitor Lambda finding patterns and automate response"
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
