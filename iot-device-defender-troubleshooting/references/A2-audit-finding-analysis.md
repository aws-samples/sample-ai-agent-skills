---
title: "A2 — Audit Finding Analysis"
description: "Diagnose and analyze IoT Device Defender audit findings"
status: active
severity: HIGH
triggers: ["audit finding", "non-compliant resource", "audit check failed", "security finding"]
owner: devops-agent
objective: "Analyze audit findings, determine severity, and guide remediation"
context: "Audit findings identify IoT configuration issues. Each finding has a check name, severity, and non-compliant resource. Findings persist until the issue is fixed and a new audit confirms resolution."
---

## Phase 1 — Triage

MUST:
- List findings for recent audit: `aws iot list-audit-findings --task-id <task-id>`
- Get finding details: `aws iot describe-audit-finding --finding-id <finding-id>`
- Check finding severity and non-compliant resource

SHOULD:
- Group findings by check name to identify patterns
- Check if findings are new or recurring from previous audits
- Review the non-compliant resource configuration

MAY:
- Check audit finding suppressions: `aws iot list-audit-suppressions`
- Compare findings across multiple audit runs

## Phase 2 — Remediate

MUST:
- Address CRITICAL and HIGH severity findings first
- Apply appropriate mitigation actions: `aws iot start-audit-mitigation-actions-task --task-id <id> --target '{"findingIds":["<finding-id>"]}' --audit-check-to-actions-mapping '{"<check>":["<action-id>"]}'`
- Verify remediation by running a new on-demand audit

SHOULD:
- Document accepted risks for findings that cannot be immediately fixed
- Create audit suppressions for known acceptable configurations

MAY:
- Set up automated mitigation for recurring findings
- Configure finding notifications via SNS

## Safety Ratings

```
safety_ratings:
  - "iot list-audit-findings: GREEN — read-only finding listing"
  - "iot describe-audit-finding: GREEN — read-only finding details"
  - "iot start-audit-mitigation-actions-task: YELLOW — applies mitigation actions to resources"
  - "iot create-audit-suppression: YELLOW — suppresses future findings for specific resources"
```

## Escalation Conditions

- CRITICAL findings indicate immediate security risks (e.g., overly permissive IoT policies)
- Large number of findings across multiple check types indicating systemic issues
- Findings cannot be remediated without affecting production device connectivity
- Recurring findings despite previous remediation attempts

## Data Sensitivity

- `aws iot list-audit-findings`: MEDIUM — reveals security configuration weaknesses
- `aws iot describe-audit-finding`: MEDIUM — contains non-compliant resource details
- `non-compliant resource details`: MEDIUM — may reveal IoT policy content and device configuration
- `mitigation action results`: LOW — returns action execution status

## Prohibited Actions

- NEVER suppress all findings without reviewing each one
- NEVER apply mitigation actions to production resources without testing
- NEVER ignore CRITICAL severity findings
- NEVER delete non-compliant resources as a quick fix without understanding impact on devices

## Phase 3 — Rollback

1. If mitigation action caused issues, check action results: `aws iot describe-audit-mitigation-actions-task --task-id <id>`
2. Reverse mitigation action effects (e.g., remove thing from quarantine group, restore policy version)
3. If audit suppression was created incorrectly, delete it: `aws iot delete-audit-suppression`
4. Re-run audit to verify current state: `aws iot start-on-demand-audit-task`
5. Document the finding analysis and remediation decisions

## Common Issues

- symptoms: "Audit finding: IOT_POLICY_OVERLY_PERMISSIVE_CHECK — policy allows iot:* on *"
  diagnosis: "IoT policy grants overly broad permissions."
  resolution: "Restrict IoT policy to specific actions and resources needed by devices."

## Output Format

```yaml
root_cause: "audit_finding — <specific_cause>"
evidence:
  - type: finding_details
    content: "<finding check name, severity, and resource>"
severity: HIGH
mitigation:
  immediate: "Apply mitigation action"
  long_term: "Fix underlying configuration and prevent recurrence"
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
