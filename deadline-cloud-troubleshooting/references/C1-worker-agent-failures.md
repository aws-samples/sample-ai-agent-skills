---
title: "C1 — Worker Agent Failures"
description: "Diagnose worker agent software failures and communication issues"
status: active
severity: CRITICAL
triggers: ["worker agent error", "agent not connecting", "worker not registering", "agent crash", "worker offline"]
owner: devops-agent
objective: "Resolve worker agent software failures to restore job execution"
context: "The worker agent is software running on fleet instances that communicates with Deadline Cloud. It manages sessions, executes tasks, and reports status. Agent failures prevent job execution."
---

## Phase 1 — Triage

MUST:
- List workers and check status: `aws deadline list-workers --farm-id <farm-id> --fleet-id <fleet-id> --query 'workers[*].{WorkerId:workerId,Status:status,CreatedAt:createdAt}'`
- Get worker details: `aws deadline get-worker --farm-id <farm-id> --fleet-id <fleet-id> --worker-id <worker-id>`
- Check worker agent logs on the instance: `/var/log/amazon/deadline/` (Linux) or `C:\ProgramData\Amazon\Deadline\Logs\` (Windows)

SHOULD:
- Verify worker agent service is running on the instance
- Check network connectivity from worker to Deadline Cloud endpoint
- Verify worker IAM role/credentials are valid

MAY:
- Check worker agent version compatibility
- Review system resource usage (CPU, memory, disk) on the worker host

## Phase 2 — Remediate

MUST:
- Restart worker agent service if crashed
- Fix network connectivity (security groups, NACLs, internet access)
- Ensure worker IAM role has required Deadline Cloud permissions

SHOULD:
- Update worker agent to latest compatible version
- Verify DNS resolution for Deadline Cloud endpoints
- Check TLS certificate validation is not blocked

MAY:
- Configure worker agent logging level for debugging
- Set up CloudWatch agent for centralized log collection

## Safety Ratings

```
safety_ratings:
  - "list-workers: GREEN — read-only worker enumeration"
  - "get-worker: GREEN — read-only worker details"
  - "Restart agent service: YELLOW — interrupts current session on the worker"
  - "Terminate worker instance: RED — kills running sessions and loses in-progress work"
```

## Escalation Conditions

- All workers in a fleet show agent failures simultaneously
- Agent failures persist after restart indicating software bug
- Network connectivity issues require VPC or firewall changes
- Agent version incompatibility requires coordinated fleet-wide update

## Data Sensitivity

- Worker agent logs: MEDIUM — contain session details and error messages
- Worker IAM credentials: HIGH — fleet execution role credentials
- Network configuration: MEDIUM — reveals connectivity architecture
- Worker host details: LOW — instance metadata

## Prohibited Actions

- NEVER terminate worker instances to fix agent issues without trying restart first
- NEVER modify worker IAM role to have admin permissions for debugging
- NEVER share worker agent logs containing session credentials
- NEVER downgrade worker agent version without testing compatibility

## Phase 3 — Rollback

1. If agent was updated and new version fails, rollback to previous version
2. If agent configuration was changed, restore previous configuration file
3. If network changes broke connectivity, revert security group or NACL changes
4. Verify workers re-register and accept new sessions after fix

## Common Issues

- symptoms: "Worker status shows NOT_RESPONDING"
  diagnosis: "Worker agent cannot communicate with Deadline Cloud service."
  resolution: "Check network connectivity. Verify security groups allow HTTPS outbound. Check DNS resolution."

- symptoms: "Worker agent crashes repeatedly on startup"
  diagnosis: "Agent configuration file corrupted or incompatible version."
  resolution: "Check agent logs for startup errors. Reinstall agent or fix configuration file."

## Output Format

```yaml
root_cause: "worker_agent — <specific_cause>"
evidence:
  - type: agent_error
    content: "<agent logs and worker status>"
severity: CRITICAL
mitigation:
  immediate: "Restart or fix agent"
  long_term: "Implement agent health monitoring and auto-recovery"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
