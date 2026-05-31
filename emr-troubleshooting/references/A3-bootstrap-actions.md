---
title: "A3 — Bootstrap Action Failures"
description: "Diagnose bootstrap action failures on EMR clusters"
status: active
severity: HIGH
triggers:
  - "bootstrap action failed"
  - "bootstrap timeout"
  - "cluster failed during bootstrap"
  - "bootstrap script error"
owner: devops-agent
objective: "Identify and resolve bootstrap action failures"
context: "Bootstrap actions run on all nodes before applications are installed. Failures terminate the cluster. Common causes: script errors, missing dependencies, S3 access issues, timeout, or permission problems. Logs are in S3 at <log-bucket>/<cluster-id>/node/<node-id>/bootstrap-actions/."
---

## Phase 1 — Triage

MUST:
- Check bootstrap action logs: `aws s3 ls s3://<log-bucket>/<cluster-id>/node/`
- List bootstrap actions: `aws emr list-bootstrap-actions --cluster-id <id>`
- Check cluster state change reason for bootstrap failure
- Download and review bootstrap action stderr/stdout from S3

SHOULD:
- Verify the bootstrap script is accessible in S3
- Check script permissions (must be executable)
- Verify script dependencies are available
- Check for timeout issues (default timeout applies)

MAY:
- Test the bootstrap script on a standalone EC2 instance
- Check for AMI compatibility issues

## Phase 2 — Remediate

MUST:
- Fix the bootstrap script errors
- Ensure the script is accessible in S3 with proper permissions
- Handle errors gracefully in the script (set -e considerations)

SHOULD:
- Add logging to bootstrap scripts for debugging
- Set appropriate timeouts for long-running scripts
- Test scripts independently before using in EMR

MAY:
- Use custom AMIs instead of bootstrap actions for complex setups
- Create reusable bootstrap action libraries

## Common Issues

- symptoms: "Bootstrap action timed out"
  diagnosis: "Script takes too long to execute."
  resolution: "Optimize script. Use custom AMI for heavy installations."

- symptoms: "Bootstrap script not found"
  diagnosis: "S3 path incorrect or permissions issue."
  resolution: "Verify S3 path and ensure instance profile can access the script."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix bootstrap script errors | GREEN | Script fix — requires cluster relaunch to apply |
| Ensure script is accessible in S3 | GREEN | Permission fix — non-destructive |
| Set appropriate timeouts | GREEN | Configuration change — non-destructive |
| Use custom AMIs instead of bootstrap actions | YELLOW | Changes cluster base image — test thoroughly |
| Relaunch cluster with fixed bootstrap | YELLOW | Creates new cluster — verify configuration |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Bootstrap failure blocks a critical data pipeline deployment
- Bootstrap script installs security-sensitive components (encryption, auth)

## Data Sensitivity

- **Classification: HIGH**
- Bootstrap scripts may contain embedded credentials or configuration secrets
- S3 log paths reveal organizational infrastructure layout
- Script dependencies reveal software stack and versions
- Bootstrap action configurations expose cluster customization patterns

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest embedding credentials directly in bootstrap scripts
- **NEVER** suggest making bootstrap scripts publicly accessible in S3

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified bootstrap script in S3 | Restore previous version of the script from S3 versioning |
| Launched new cluster with fixed bootstrap | Terminate the new cluster if bootstrap fix is incorrect |
| Switched to custom AMI | Revert to previous AMI or bootstrap-based approach |

## Output Format

```yaml
root_cause: "bootstrap_failure — <specific_cause>"
evidence:
  - type: bootstrap_logs
    content: "<bootstrap action stderr/stdout>"
  - type: script_config
    content: "<bootstrap action configuration>"
severity: HIGH
mitigation:
  immediate: "Fix bootstrap script and relaunch cluster"
  long_term: "Test bootstrap scripts independently and use custom AMIs for complex setups"
```

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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
