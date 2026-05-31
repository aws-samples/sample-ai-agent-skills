# C3 — SMS Instance Launch Issues

## Symptoms
- Migrated instance won't boot
- Instance launches but application doesn't work
- Network configuration wrong after launch

## Triage Commands

```bash
aws ec2 describe-instances --filters "Name=tag:aws:sms:*,Values=*" --max-results 10
aws ec2 get-console-output --instance-id <instance-id>
aws ec2 describe-instance-status --instance-ids <instance-id>
```

## Common Causes
1. **Boot configuration** — GRUB/bootloader not EC2-compatible
2. **Network drivers** — Missing AWS network drivers (ENA/ixgbevf)
3. **DHCP** — Static IP config from source not working in VPC
4. **Security groups** — Ports not open for application
5. **DNS** — Hostname resolution different in AWS

## Resolution
- Check console output for boot errors: `aws ec2 get-console-output`
- For Windows: install AWS PV drivers before migration
- For Linux: ensure DHCP is enabled on primary interface
- Configure security groups to allow application ports
- Update DNS/hostname configuration for AWS environment
- Test with a non-production instance before production cutover


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

- Server catalog reveals on-premises infrastructure details
- Replication data contains VM disk contents
- Migration configurations reveal network topology
- Classification: CONFIDENTIAL - infrastructure details and VM data

## Prohibited Actions

- NEVER delete replication jobs with in-progress migrations
- NEVER modify server catalog mappings during active replication
- NEVER terminate migration runs without confirming rollback capability
- NEVER remove SMS connectors while replication jobs are active

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
