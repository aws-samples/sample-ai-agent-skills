# B1 — Agentless Collector Issues

## Symptoms
- Collector OVA deployment fails
- Collector not discovering VMs
- Collector status errors

## Triage Commands

```bash
aws discovery describe-continuous-exports --max-results 10
aws discovery get-discovery-summary
aws discovery list-configurations --configuration-type SERVER --max-results 20
```

## Common Causes

1. **OVA deployment failed** — VMware compatibility or resource issues
2. **vCenter credentials wrong** — Can't authenticate to vCenter API
3. **Network connectivity** — Collector can't reach AWS endpoints
4. **vCenter permissions** — Insufficient vCenter privileges
5. **Collector not configured** — Deployed but not set up via web UI

## Resolution

- Deploy OVA in VMware vSphere 6.0+ environment
- Configure collector via its web UI (https://<collector-ip>)
- Provide vCenter credentials with read-only access
- Allow outbound HTTPS to AWS endpoints
- Verify vCenter user has at least read-only role on datacenter


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Deploy OVA in vSphere | YELLOW | Infrastructure deployment |
| Configure collector via web UI | GREEN | Configuration only |
| Provide vCenter credentials | YELLOW | Credential sharing with collector |
| Allow outbound HTTPS | YELLOW | Network configuration change |
| Verify vCenter read-only role | GREEN | Permission verification |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **vCenter credentials**: Admin or read-only credentials — CONFIDENTIAL
- **VM inventory**: Full list of virtual machines — INTERNAL
- **Network endpoints**: AWS endpoint connectivity — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop discovery collection from the collector web UI
2. Disconnect vCenter integration from the collector
3. Delete the collector OVA from the vSphere environment
4. Revoke any credentials shared with the collector
5. Verify no data is being collected: `aws discovery get-discovery-summary`

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
