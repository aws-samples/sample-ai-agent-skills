---
title: "D1 — Patch Baseline Configuration"
description: "Diagnose patch baseline configuration and assignment issues"
status: active
severity: MEDIUM
triggers:
  - "patch baseline not working"
  - "wrong patches applied"
  - "baseline configuration"
  - "patch group assignment"
owner: devops-agent
objective: "Identify patch baseline configuration issues and ensure correct patch assignment"
context: "Patch baselines define which patches are approved for installation. Each baseline targets one OS. Patch groups map instances to baselines. AWS provides default baselines per OS. Custom baselines allow control over classification, severity, and auto-approval delays."
---

## Phase 1 — Triage

MUST:
- List patch baselines: `aws ssm describe-patch-baselines --filters Key=OWNER,Values=Self`
- Get baseline details: `aws ssm get-patch-baseline --baseline-id <baseline-id>`
- Check default baselines: `aws ssm describe-patch-baselines --filters Key=OWNER,Values=AWS`
- List patch groups: `aws ssm describe-patch-groups`

SHOULD:
- Check effective patches for baseline: `aws ssm describe-effective-patches-for-patch-baseline --baseline-id <baseline-id> --max-results 50`
- Verify patch group registration: `aws ssm get-patch-baseline-for-patch-group --patch-group <group-name>`
- Check instance patch group tag: `aws ec2 describe-tags --filters Name=resource-id,Values=<instance-id> Name=key,Values=Patch Group`

MAY:
- Compare custom baseline with AWS default baseline
- Check patch baseline approval rules and exceptions

## Phase 2 — Remediate

MUST:
- Ensure baseline targets the correct operating system
- Register patch groups to the appropriate baseline
- Tag instances with the correct Patch Group value

SHOULD:
- Define clear approval rules (classification, severity, auto-approval delay)
- Test baselines in a non-production patch group first
- Document baseline configurations and patch group assignments

MAY:
- Create separate baselines for different environments (dev, staging, prod)
- Use patch exceptions for specific KB articles or CVEs

## Common Issues

- symptoms: "Patches not being applied to instances"
  diagnosis: "Instance not in a patch group or patch group not registered to baseline."
  resolution: "Tag instance with 'Patch Group' tag and register group to baseline."

- symptoms: "Wrong patches being approved"
  diagnosis: "Baseline approval rules too broad or auto-approval delay too short."
  resolution: "Review and tighten approval rules. Increase auto-approval delay."

- symptoms: "Baseline shows 0 effective patches"
  diagnosis: "Baseline OS does not match target instances or rules too restrictive."
  resolution: "Verify baseline OS matches instances. Review classification and severity filters."

## Output Format

```yaml
root_cause: "patch_baseline — <specific_cause>"
evidence:
  - type: baseline_config
    content: "<get-patch-baseline output>"
  - type: patch_groups
    content: "<patch group assignments>"
severity: MEDIUM
mitigation:
  immediate: "Fix baseline configuration or patch group assignment"
  long_term: "Implement baseline testing and change management process"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Parameter Store may contain secrets, API keys, and credentials
- Command output may contain system configuration details
- Session logs may capture sensitive terminal activity
- Classification: HIGHLY CONFIDENTIAL - secrets management and system access data

## Prohibited Actions

- NEVER execute untested commands via Run Command on production instances
- NEVER modify patch baselines during active patching windows
- NEVER delete SSM parameters containing production secrets without rotation
- NEVER deregister managed instances without confirming no active sessions

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
