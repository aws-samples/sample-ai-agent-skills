---
title: "Z1 — General Troubleshooting"
description: "General CloudShell troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["CloudShell error", "CloudShell issue", "CloudShell not working", "CloudShell problem"]
owner: devops-agent
objective: "Diagnose and resolve general CloudShell issues using systematic investigation"
context: "Catch-all runbook for CloudShell issues. Start here when the specific problem category is unclear."
---

## Phase 1 — Triage

MUST:
- Identify the symptom category: launch, network, storage, permissions, timeout, or performance
- Check CloudShell environment status: `aws cloudshell describe-environments`
- Verify IAM identity and permissions: `aws sts get-caller-identity`

SHOULD:
- Review recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudshell.amazonaws.com --max-results 20`
- Check AWS Health Dashboard for CloudShell service issues
- Verify region supports CloudShell

MAY:
- Test basic operations: create file, run AWS CLI command, check network
- Compare behavior across different browsers or regions

## Phase 2 — Remediate

MUST:
- Route to specific runbook once symptom category is identified:
  - Launch issues → A1-session-launch-failures
  - Persistence → A2-environment-persistence
  - Network → B1-network-connectivity or B2-vpc-environment
  - Storage → C1-storage-issues or C2-file-transfer
  - Permissions → D1-permissions-errors or D2-credential-issues
  - Timeouts → E1-timeout-disconnection
  - Performance → E2-performance-issues
  - Customization → F1-custom-environment

SHOULD:
- Document the issue for pattern recognition
- Check if issue is reproducible across sessions

MAY:
- Open AWS Support case if issue persists after runbook remediation
- Check AWS forums and re:Post for similar reported issues

## Safety Ratings

```
safety_ratings:
  - "describe-environments: GREEN — read-only environment listing"
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "cloudtrail lookup-events: GREEN — read-only event history"
  - "delete-environment: RED — permanently removes environment and data"
```

## Escalation Conditions

- Issue persists after following specific runbook remediation steps
- Multiple users affected simultaneously (possible service issue)
- Issue involves AWS service degradation or outage
- Security-related issue requiring immediate attention

## Data Sensitivity

- `aws cloudshell describe-environments`: LOW — environment metadata
- `aws sts get-caller-identity`: LOW — account and identity info
- `aws cloudtrail lookup-events`: MEDIUM — user activity records
- Browser console logs: LOW — connection metadata

## Prohibited Actions

- NEVER delete environments without confirming data backup
- NEVER escalate to AWS Support without first following relevant runbook
- NEVER assume the issue is CloudShell-specific without checking IAM and network
- NEVER share CloudShell session details containing sensitive information

## Phase 3 — Rollback

1. If any remediation step made the issue worse: revert the specific change
2. If environment was deleted: create new environment and restore from backup
3. If IAM changes were made: revert to original policy configuration
4. If network changes were made: restore original routing and security group rules
5. Document all changes made during troubleshooting

## Common Issues

- symptoms: "CloudShell shows a spinning wheel and never loads"
  diagnosis: "Browser WebSocket connection blocked, or CloudShell service issue in the region."
  resolution: "Try a different browser, disable VPN/proxy, or switch to a different AWS region."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Apply specific runbook remediation"
  long_term: "Document issue pattern and implement preventive measures"
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
