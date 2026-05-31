# F1 — SMS Replication Scheduling Issues

## Symptoms
- Replication not running on schedule
- Schedule configuration errors
- Missed replication windows

## Triage Commands

```bash
aws sms get-replication-jobs --max-results 20
aws sms get-replication-runs --replication-job-id <job-id> --max-results 10
```

## Replication Frequencies
| Frequency | Use Case |
|-----------|----------|
| 1 hour | Low RPO, high change rate |
| 6 hours | Moderate change rate |
| 12 hours | Low change rate |
| 24 hours | Daily sync |

## Common Causes
1. **Previous run still active** — Next run waits for current to finish
2. **Connector busy** — Multiple jobs competing for connector resources
3. **Schedule not set** — Frequency not configured in job
4. **Job paused** — Replication job manually paused
5. **90-day expiry** — Job expired and stopped scheduling

## Resolution
- Check if previous run is still in progress
- Reduce concurrent replication jobs if connector is overloaded
- Verify frequency setting in replication job configuration
- Resume paused jobs: recreate if expired
- For final sync before cutover, trigger manual run


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
