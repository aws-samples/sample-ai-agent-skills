# B1 — SMS Server Catalog Issues

## Symptoms
- Server catalog empty
- Servers missing from catalog
- Catalog import fails

## Triage Commands

```bash
aws sms get-servers --max-results 50
aws sms import-server-catalog
aws sms get-connectors --max-results 10
```

## Common Causes
1. **Connector not connected to vCenter** — Can't discover VMs
2. **Import not run** — Must explicitly import catalog
3. **vCenter permissions** — Connector user needs read access
4. **Connector version** — Old connector version
5. **Large inventory** — Import takes time for many VMs

## Resolution
- Import catalog: `aws sms import-server-catalog`
- Verify connector has vCenter credentials configured
- Check connector can reach vCenter API
- Wait for import to complete (check with `get-servers`)
- Re-import periodically to catch new VMs


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
