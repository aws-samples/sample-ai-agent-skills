---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for EFA issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["efa issue", "efa error", "efa problem", "general efa"]
owner: devops-agent
objective: "Provide a systematic approach for diagnosing unclassified EFA issues"
context: "Use this runbook when the issue does not clearly match a specific category. Follow the triage steps to classify and route to the appropriate runbook."
---

## Phase 1 — Triage

MUST:
- Verify EFA interface is attached: `aws ec2 describe-network-interfaces --filters "Name=attachment.instance-id,Values=<instance-id>" "Name=interface-type,Values=efa"`
- Check EFA driver status: `lsmod | grep efa && fi_info -p efa`
- Verify security group self-referencing rule: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check placement group configuration: `aws ec2 describe-instances --instance-ids <id> --query 'Reservations[0].Instances[0].Placement'`
- Verify instance type supports EFA: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].NetworkInfo.EfaSupported'`

MAY:
- Run comprehensive EFA diagnostics: `fi_info -p efa -v` and `ibv_devinfo`
- Check dmesg for EFA-related kernel messages

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- If creation/attachment issue → A1 or A2
- If driver/library issue → B1 or B2
- If MPI/NCCL issue → C1 or C2
- If performance/bandwidth → D1 or D2
- If security group/placement → E1 or E2
- If SageMaker → F1

SHOULD:
- Document the symptoms and triage findings before routing
- Verify the complete EFA stack: interface → driver → libfabric → application

MAY:
- Check AWS Health Dashboard for EFA-related service issues

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-network-interfaces: GREEN — read-only interface query"
  - "fi_info -p efa: GREEN — read-only provider check"
  - "lsmod: GREEN — read-only kernel module listing"
  - "ibv_devinfo: GREEN — read-only device information"
```

## Escalation Conditions

- Issue cannot be classified into any specific runbook category
- Multiple symptoms spanning different categories simultaneously
- Suspected AWS infrastructure issue affecting EFA in the region
- EFA interface shows errors in device statistics

## Data Sensitivity

- EFA interface details: LOW — network interface metadata
- Driver and library versions: LOW — software version information
- Security group rules: MEDIUM — network access configuration
- Kernel messages: MEDIUM — may contain system configuration details

## Prohibited Actions

- NEVER terminate instances as a first troubleshooting step — collect diagnostics first
- NEVER apply multiple changes simultaneously — change one thing at a time
- NEVER skip the security group self-referencing check — it is the most common issue
- NEVER assume EFA is working just because the interface is attached

## Phase 3 — Rollback

1. Revert any changes made during investigation
2. If driver was modified, reboot to restore previous state
3. If security group was changed, restore original rules
4. Document all changes made and their outcomes
5. If instance is unrecoverable, terminate and relaunch with correct configuration

## Common Issues

- symptoms: "EFA interface is attached but fi_info shows no EFA provider"
  diagnosis: "EFA driver is not installed or not loaded."
  resolution: "Install EFA driver package and load module: sudo modprobe efa."

- symptoms: "Everything appears configured correctly but EFA communication fails"
  diagnosis: "Security group is missing the self-referencing all-traffic inbound rule."
  resolution: "Add: aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol -1 --source-group <sg-id>."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: diagnostic_finding
    content: "<finding details>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement EFA health monitoring"
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
