---
title: "A2 — Agent Installation Issues"
description: "Diagnose AWS Replication Agent installation failures"
status: active
severity: HIGH
triggers:
  - "agent install failed"
  - "replication agent error"
  - "agent not connecting"
owner: devops-agent
objective: "Resolve AWS Replication Agent installation and connectivity issues"
context: "The AWS Replication Agent must be installed on source servers. It requires specific OS versions, kernel compatibility (Linux), outbound network access, and valid IAM credentials."
---

## Phase 1 — Triage

MUST:
- Verify OS is in the DRS supported OS list
- Check agent installer exit code and logs
- On Linux: check /var/log/aws-replication-agent/aws-replication-agent.log
- On Windows: check C:\Program Files (x86)\AWS Replication Agent\logs\

SHOULD:
- Verify kernel version compatibility (Linux): `uname -r`
- Check disk space for agent installation (minimum 2GB free)
- Verify no conflicting software (other replication agents)

MAY:
- Test manual connectivity: `curl -v https://drs.<region>.amazonaws.com`
- Check if SELinux or AppArmor is blocking agent

## Phase 2 — Remediate

MUST:
- Install supported kernel version if current kernel is incompatible
- Provide valid IAM credentials (access key or instance profile)
- Ensure outbound TCP 443 and TCP 1500 connectivity

SHOULD:
- Uninstall conflicting replication software before installing DRS agent
- Update OS packages to supported versions

MAY:
- Configure agent proxy settings for restricted networks

## Common Issues

- symptoms: "Agent fails with kernel not supported"
  diagnosis: "Linux kernel version not in DRS supported list."
  resolution: "Update kernel to a supported version. Check DRS docs for supported kernels."

- symptoms: "Agent connects but replication doesn't start"
  diagnosis: "TCP 1500 to replication server blocked."
  resolution: "Open TCP 1500 outbound from source to staging area subnet."

## Output Format

```yaml
root_cause: "agent_installation — <specific_cause>"
evidence:
  - type: agent_logs
    content: "<agent log errors>"
  - type: os_compatibility
    content: "<OS and kernel version check>"
severity: HIGH
mitigation:
  immediate: "Fix the specific installation blocker"
  long_term: "Pre-validate OS compatibility before agent deployment"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Agent installation requires IAM credentials and system access |
| Blast Radius | LOW — Agent installation doesn't affect existing server operations |
| Rollback Complexity | LOW — Agent can be uninstalled |
| Compliance Scope | HIGH — Agent installation is required for DR protection |

## Escalation Conditions

- Escalate to server owners if OS compatibility issues prevent installation
- Escalate to security if agent requires elevated privileges
- Escalate to network team if firewall rules need modification

## Data Sensitivity

- **IAM credentials** provided to the agent are authentication secrets
- **Agent logs** may contain server configuration details
- **OS and kernel information** is infrastructure data

## PROHIBITED Actions

- Do NOT install agents on servers without server owner authorization
- Do NOT use root/admin IAM credentials for the agent
- Do NOT leave agent installer files with embedded credentials on the server
- Do NOT skip OS compatibility checks before installation

## Phase 3 — Rollback

- Uninstall the agent from the source server
- Remove IAM credentials that were created for the agent
- Revert any firewall rule changes made for agent connectivity

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
