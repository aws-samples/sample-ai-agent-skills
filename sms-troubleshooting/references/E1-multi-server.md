# E1 — SMS Multi-Server Coordination Issues

## Symptoms
- Servers replicating at different rates
- Coordinated cutover timing issues
- Some servers fail while others succeed

## Triage Commands

```bash
aws sms get-app --app-id <app-id>
aws sms get-app-replication-configuration --app-id <app-id>
aws sms get-replication-jobs --max-results 50
```

## Common Causes
1. **Different replication frequencies** — Servers on different schedules
2. **Large disk variance** — Big disks take longer to replicate
3. **Network bandwidth** — Connector bandwidth shared across all jobs
4. **Staggered start times** — Jobs started at different times
5. **Individual failures** — One server failure doesn't stop others

## Resolution
- Align replication frequencies across all servers in the application
- Schedule final replication run for all servers before cutover
- Monitor all replication jobs for completion before launching
- Plan cutover window based on largest server's replication time
- Use application-level launch for coordinated instance creation


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
