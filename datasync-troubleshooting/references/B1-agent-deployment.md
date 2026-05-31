---
title: "B1 — Agent Deployment Failures"
description: "Diagnose DataSync agent deployment and activation issues"
status: active
severity: HIGH
triggers:
  - "agent deployment failed"
  - "agent activation failed"
  - "cannot activate agent"
  - "agent not appearing"
owner: devops-agent
objective: "Identify why a DataSync agent cannot be deployed or activated and resolve the issue"
context: "DataSync agents are standalone VMs deployed on VMware ESXi, Hyper-V, KVM, or as EC2 instances. Activation requires the agent to be reachable on port 80 from the browser and have outbound access on port 443 to the DataSync service. Common failures include incorrect VM sizing, network configuration, and activation endpoint issues."
---

## Phase 1 — Triage

MUST:
- Verify agent VM meets minimum requirements: 4 vCPUs, 16 GB RAM, 80 GB disk
- Check agent VM network configuration: IP address, DNS, default gateway
- Test agent activation endpoint: `curl http://<agent-ip>/?activationRegion=<region>&redirect_type=JSON`
- Verify outbound connectivity on port 443 from agent VM

SHOULD:
- Check hypervisor compatibility (ESXi 6.5+, Hyper-V 2012 R2+, KVM)
- Verify NTP synchronization on agent VM (time skew causes activation failures)
- Check if a firewall is blocking port 80 (activation) or port 443 (service communication)

MAY:
- Check agent VM console for boot errors
- Verify the DataSync service endpoint is reachable: `nslookup datasync.<region>.amazonaws.com`

## Phase 2 — Remediate

MUST:
- Ensure agent VM has correct resources and network configuration
- Open port 80 inbound (temporarily for activation) and port 443 outbound
- Activate agent: `aws datasync create-agent --activation-key <key> --agent-name <name>`

SHOULD:
- Configure NTP on the agent VM to sync with a reliable time source
- Use VPC endpoints for DataSync if the agent is in a private subnet: `aws ec2 create-vpc-endpoint --vpc-id <vpc-id> --service-name com.amazonaws.<region>.datasync`

MAY:
- Deploy agent as EC2 instance for simpler networking
- Set up monitoring for agent health after activation

## Common Issues

- symptoms: "Cannot reach agent activation endpoint on port 80"
  diagnosis: "Firewall or security group blocking port 80 inbound to agent."
  resolution: "Open port 80 inbound temporarily for activation. Can be closed after activation."

- symptoms: "Activation key generation fails"
  diagnosis: "Agent cannot reach DataSync service on port 443."
  resolution: "Check outbound connectivity. Verify security groups, NACLs, and route tables."

- symptoms: "Agent activation times out"
  diagnosis: "Time skew between agent VM and AWS, or DNS resolution failure."
  resolution: "Sync NTP on agent VM. Verify DNS resolves DataSync service endpoint."

## Output Format

```yaml
root_cause: "agent_deployment_failure — <specific_cause>"
evidence:
  - type: vm_resources
    content: "<agent VM CPU, RAM, disk>"
  - type: network_config
    content: "<IP, DNS, gateway, port connectivity>"
  - type: activation_result
    content: "<activation attempt result>"
severity: HIGH
mitigation:
  immediate: "Fix the specific deployment or activation blocker"
  long_term: "Document agent deployment requirements and automate with IaC"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Agent deployment involves network configuration and credentials |
| Blast Radius | LOW — Agent deployment does not affect existing data |
| Rollback Complexity | LOW — Agent can be deactivated and redeployed |
| Compliance Scope | MEDIUM — Agent must meet security requirements for the environment |

## Escalation Conditions

- Escalate to network team if firewall rules prevent agent activation
- Escalate to security if agent requires credentials that exceed least-privilege
- Escalate to infrastructure team if hypervisor compatibility issues arise

## Data Sensitivity

- **Agent activation keys** are temporary but should not be shared
- **Network configuration** (IPs, ports, VPC endpoints) reveals infrastructure topology
- **Agent VM credentials** must be secured per infrastructure policy

## PROHIBITED Actions

- Do NOT leave port 80 open on the agent after activation is complete
- Do NOT deploy agents with default or weak VM credentials
- Do NOT skip NTP configuration — time skew causes activation failures and security issues
- Do NOT deploy agents in untrusted network segments

## Phase 3 — Rollback

- Delete the failed agent: `aws datasync delete-agent --agent-arn <arn>`
- Close any firewall rules opened for activation (port 80)
- Redeploy the agent VM with corrected configuration
- Re-activate with proper network and credential settings

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
