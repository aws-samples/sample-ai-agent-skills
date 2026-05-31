# A1 — Application Discovery Agent Deployment Issues

## Symptoms
- Agent installation fails
- Agent not registering with service
- Agent not appearing in console

## Triage Commands

```bash
aws discovery describe-agents --max-results 20
aws discovery describe-agents --filters name=agentIds,values=<agent-id>,condition=EQUALS
aws mgh describe-home-region
```

## Common Causes

1. **Home region not set** — Must set Migration Hub home region first
2. **OS not supported** — Agent supports specific OS versions only
3. **No outbound HTTPS** — Agent needs port 443 to discovery endpoint
4. **IAM credentials** — Agent needs valid AWS credentials for registration
5. **Proxy not configured** — Corporate proxy blocking outbound connections

## Resolution

- Set home region: `aws mgh create-home-region-control --home-region <region>`
- Verify OS compatibility (Amazon Linux 2, Ubuntu 14+, RHEL 6+, Windows Server 2008+)
- Allow outbound HTTPS to `discovery.<home-region>.amazonaws.com`
- Configure agent with valid IAM access key/secret key
- For proxy: configure proxy settings in agent configuration file


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Set home region | GREEN | Non-destructive configuration |
| Verify OS compatibility | GREEN | Read-only check |
| Allow outbound HTTPS | YELLOW | Network configuration change |
| Configure agent with IAM credentials | YELLOW | Credential configuration on host |
| Configure proxy settings | YELLOW | Agent configuration change |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Server metadata**: Hostnames, IP addresses, OS versions — INTERNAL
- **IAM credentials**: Access keys used by agents — CONFIDENTIAL
- **Network topology**: Connectivity data reveals infrastructure layout — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop data collection: `aws discovery stop-data-collection-by-agent-ids --agent-ids <id>`
2. Uninstall the agent from the affected server using OS package manager
3. Revoke IAM credentials used by the agent: `aws iam delete-access-key --user-name <user> --access-key-id <key-id>`
4. Verify agent is no longer reporting: `aws discovery describe-agents --filters name=agentIds,values=<agent-id>,condition=EQUALS`
5. Remove any firewall rules added for agent communication

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
