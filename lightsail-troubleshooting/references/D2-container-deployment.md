---
title: "D2 — Container Deployment Issues"
description: "Diagnose container deployment failures and image problems"
status: active
severity: HIGH
triggers:
  - "deployment failed"
  - "image pull error"
  - "container deployment"
  - "deployment stuck"
owner: devops-agent
objective: "Identify and resolve container deployment failures"
context: "Container deployments create new versions of the container service. Deployments can fail due to image pull errors, health check failures, resource constraints, or configuration errors. Failed deployments automatically roll back to the previous version."
---

## Phase 1 — Triage

MUST:
- Check deployment status: `aws lightsail get-container-service-deployments --service-name <service-name>`
- Check container logs for the failing deployment
- Verify container image exists and is accessible
- Check health check configuration

SHOULD:
- Verify container image was pushed to Lightsail registry: `aws lightsail get-container-images --service-name <service-name>`
- Check if deployment rolled back to previous version
- Verify environment variables and port configuration

MAY:
- Test container image locally before deploying
- Review deployment configuration JSON

## Phase 2 — Remediate

MUST:
- Fix container image issues (build errors, missing dependencies)
- Ensure health check endpoint is implemented and returns 200
- Push fixed image and redeploy

SHOULD:
- Push image to Lightsail registry: `aws lightsail push-container-image --service-name <name> --label <label> --image <local-image>`
- Test locally before deploying to Lightsail

MAY:
- Use multi-stage Docker builds to reduce image size
- Implement graceful shutdown handling in containers

## Common Issues

- symptoms: "Deployment fails with image pull error"
  diagnosis: "Image not found in registry or public registry inaccessible."
  resolution: "Push image to Lightsail registry. Verify image name and tag."

- symptoms: "Deployment fails health check"
  diagnosis: "Application not responding on health check path within timeout."
  resolution: "Implement health check endpoint. Ensure app starts within timeout."

- symptoms: "Deployment succeeds but app not working"
  diagnosis: "Wrong port configuration or environment variables."
  resolution: "Verify container port matches public endpoint port. Check env vars."

## Output Format

```yaml
root_cause: "container_deployment — <specific_cause>"
evidence:
  - type: deployment_status
    content: "<deployment state and version>"
  - type: image_status
    content: "<container image availability>"
  - type: health_check
    content: "<health check configuration and result>"
severity: HIGH
mitigation:
  immediate: "Fix image/config and redeploy"
  long_term: "Implement CI/CD pipeline with pre-deployment testing"
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
