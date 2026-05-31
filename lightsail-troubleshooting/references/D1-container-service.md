---
title: "D1 — Container Service Issues"
description: "Diagnose Lightsail container service creation and operational problems"
status: active
severity: HIGH
triggers:
  - "container service error"
  - "container not running"
  - "container service failed"
  - "container health check"
owner: devops-agent
objective: "Identify and resolve Lightsail container service issues"
context: "Lightsail container services run Docker containers with predefined power levels (Nano to XLarge) and 1-20 nodes. Services have a public HTTPS endpoint. Container images can come from Lightsail registry or public registries. Health checks determine container readiness."
---

## Phase 1 — Triage

MUST:
- Check service status: `aws lightsail get-container-services --service-name <service-name>`
- Check current deployment: `aws lightsail get-container-service-deployments --service-name <service-name>`
- Check container logs: `aws lightsail get-container-log --service-name <service-name> --container-name <container-name>`
- Verify service power and scale configuration

SHOULD:
- Check health check configuration and results
- Verify container image is accessible
- Review deployment version history

MAY:
- Check container service metrics
- Review public endpoint accessibility

## Phase 2 — Remediate

MUST:
- Fix container image or configuration issues
- Ensure health check endpoint returns HTTP 200
- Redeploy after fixing: `aws lightsail create-container-service-deployment --service-name <name> --containers <config> --public-endpoint <endpoint-config>`

SHOULD:
- Scale up power level if resource-constrained
- Increase node count for availability

MAY:
- Use Lightsail container registry for private images
- Implement proper health check endpoints in application

## Common Issues

- symptoms: "Container service stuck in DEPLOYING"
  diagnosis: "Container failing health checks or image pull errors."
  resolution: "Check container logs. Verify health check endpoint. Check image accessibility."

- symptoms: "Container keeps restarting"
  diagnosis: "Application crashing or health check failing."
  resolution: "Check container logs for crash reason. Fix application or health check."

- symptoms: "Public endpoint returning 503"
  diagnosis: "No healthy containers available."
  resolution: "Check container health. Fix application issues. Verify health check path."

## Output Format

```yaml
root_cause: "container_service — <specific_cause>"
evidence:
  - type: service_status
    content: "<container service state>"
  - type: deployment_status
    content: "<current deployment state>"
  - type: container_logs
    content: "<recent container log entries>"
severity: HIGH
mitigation:
  immediate: "Fix container configuration and redeploy"
  long_term: "Implement proper health checks and monitoring"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a static IP address attached to a production instance without reassignment planning
- **NEVER** modify firewall rules to allow unrestricted inbound access (0.0.0.0/0) on sensitive ports
- **NEVER** delete snapshots that serve as the only backup for production instances or databases
- **NEVER** force-stop a managed database instance without verifying active connections and transactions
- **NEVER** detach a disk from a running instance without first unmounting it from the operating system

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
