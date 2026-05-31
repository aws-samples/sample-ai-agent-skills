---
title: "Z1 — General Backup Gateway Troubleshooting"
description: "Catch-all runbook for Backup Gateway issues"
status: active
severity: MEDIUM
triggers: ["Backup Gateway error", "backup gateway not working"]
owner: devops-agent
objective: "Systematic investigation of unclassified Backup Gateway issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check gateway status: `aws backup-gateway list-gateways --query 'Gateways[*].{Name:GatewayDisplayName,Arn:GatewayArn,LastSeen:LastSeenTime}'`
- Check hypervisor connections: `aws backup-gateway list-hypervisors --query 'Hypervisors[*].{Name:Name,State:State}'`
- Review recent backup jobs: `aws backup list-backup-jobs --by-resource-type VIRTUALMACHINE --max-results 10`

SHOULD:
- Verify AWS credentials: `aws sts get-caller-identity`
- Check CloudTrail for Backup Gateway events

MAY:
- Check AWS Health Dashboard for service issues
- Review CloudWatch metrics for the gateway

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for recurring issues

## Safety Ratings

```
safety_ratings:
  - "backup-gateway list-gateways: GREEN — read-only gateway listing"
  - "backup-gateway list-hypervisors: GREEN — read-only hypervisor listing"
  - "backup list-backup-jobs: GREEN — read-only job listing"
  - "sts get-caller-identity: GREEN — read-only identity check"
```

## Escalation Conditions

- Backup Gateway service appears non-functional
- All gateways are offline simultaneously
- Multiple backup jobs fail across different VMs
- Issue cannot be classified into any specific runbook

## Data Sensitivity

- `aws backup-gateway list-gateways`: LOW — returns gateway metadata
- `aws backup-gateway list-hypervisors`: MEDIUM — reveals hypervisor hosts
- `aws backup list-backup-jobs`: MEDIUM — reveals backup job history

## Prohibited Actions

- NEVER delete gateways during general troubleshooting
- NEVER make configuration changes without identifying root cause
- NEVER dismiss the issue without documenting the investigation
- NEVER assume gateway offline means backup data is lost

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Verify all gateways are online and connected
3. Check that hypervisor connections are active
4. Document all investigation steps and findings
5. Verify Backup Gateway is functioning normally before closing

## Common Issues

- symptoms: "Backup Gateway is not working — no specific error"
  diagnosis: "Multiple possible causes — gateway connectivity, hypervisor connection, or backup configuration."
  resolution: "Follow triage steps: check gateway status, hypervisor connections, and recent backup jobs. Pivot to specific runbook."

## Output Format

```yaml
root_cause: "general_backup_gateway — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
