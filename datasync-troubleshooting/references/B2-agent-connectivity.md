---
title: "B2 — Agent Connectivity Issues"
description: "Diagnose DataSync agent connectivity and offline problems"
status: active
severity: HIGH
triggers:
  - "agent offline"
  - "agent connectivity"
  - "agent not responding"
  - "agent status OFFLINE"
owner: devops-agent
objective: "Identify why a DataSync agent is offline or has connectivity issues and restore connectivity"
context: "DataSync agents maintain a persistent connection to the DataSync service over port 443. Agents go OFFLINE when this connection is lost due to network issues, VM problems, or service endpoint changes. An offline agent cannot execute tasks."
---

## Phase 1 — Triage

MUST:
- Check agent status: `aws datasync describe-agent --agent-arn <agent-arn>`
- Verify agent VM is running and accessible
- Test outbound connectivity from agent to DataSync service on port 443
- Check security groups and NACLs for port 443 outbound

SHOULD:
- Check agent VM resource utilization (CPU, memory, disk full)
- Verify DNS resolution for DataSync service endpoint
- Check if VPC endpoint for DataSync is healthy (if applicable): `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.datasync`

MAY:
- Review agent VM system logs for errors
- Check if recent network changes affected connectivity

## Phase 2 — Remediate

MUST:
- Restore network connectivity on port 443 outbound
- Restart agent VM if it is unresponsive
- Verify agent returns to ONLINE status after fix

SHOULD:
- Check agent status after restart: `aws datasync describe-agent --agent-arn <agent-arn>`
- Verify pending tasks resume after agent comes online

MAY:
- Set up CloudWatch alarms for agent status changes
- Configure redundant network paths for agent connectivity

## Common Issues

- symptoms: "Agent shows OFFLINE in console"
  diagnosis: "Agent VM lost network connectivity to DataSync service."
  resolution: "Check security groups, NACLs, route tables for port 443 outbound. Restart agent VM."

- symptoms: "Agent intermittently goes offline"
  diagnosis: "Unstable network connection or agent VM resource exhaustion."
  resolution: "Check network stability. Verify agent VM has sufficient resources. Check for disk full."

- symptoms: "Agent offline after VPC changes"
  diagnosis: "Route table or security group changes broke connectivity."
  resolution: "Review recent VPC changes. Ensure route to DataSync endpoint exists."

## Output Format

```yaml
root_cause: "agent_connectivity — <specific_cause>"
evidence:
  - type: agent_status
    content: "<agent describe output>"
  - type: network_test
    content: "<port 443 connectivity test result>"
  - type: vm_health
    content: "<agent VM resource utilization>"
severity: HIGH
mitigation:
  immediate: "Restore agent connectivity and verify ONLINE status"
  long_term: "Set up monitoring and alerting for agent status"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Agent connectivity issues may expose network architecture details |
| Blast Radius | HIGH — Offline agent stops all replication tasks using that agent |
| Rollback Complexity | LOW — Restore connectivity and agent comes back online |
| Compliance Scope | HIGH — Agent downtime may violate data transfer SLAs |

## Escalation Conditions

- Escalate IMMEDIATELY to network team if agent goes offline unexpectedly
- Escalate to operations if agent downtime exceeds SLA thresholds
- Escalate to AWS Support if agent remains offline after connectivity is restored

## Data Sensitivity

- **Agent status** reveals operational health — operational data
- **Network diagnostics** may expose VPC and security group configurations

## PROHIBITED Actions

- Do NOT modify security groups or NACLs without understanding the impact on all services
- Do NOT ignore intermittent agent offline events — they indicate underlying network instability
- Do NOT restart the agent VM without checking for active task executions

## Phase 3 — Rollback

- Restore the network path (security groups, NACLs, route tables) to previous working state
- Restart the agent VM if it is unresponsive
- Verify agent returns to ONLINE status: `aws datasync describe-agent --agent-arn <arn>`
- Check that pending tasks resume after agent recovery

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
