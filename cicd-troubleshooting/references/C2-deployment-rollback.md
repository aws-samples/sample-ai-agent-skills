---
title: "C2 — Deployment Rollback"
description: "Diagnose and resolve CodeDeploy rollback issues"
status: active
severity: HIGH
triggers:
  - "rollback"
  - "rolled back"
  - "auto rollback"
  - "rollback failed"
  - "previous revision"
owner: devops-agent
objective: "Identify why rollbacks are occurring or failing and resolve the issue"
context: "CodeDeploy supports automatic rollback on deployment failure or CloudWatch alarm breach. Rollback behavior differs between in-place (redeploys previous revision) and blue/green (reroutes traffic to original instances). Rollback failures can leave the deployment in an inconsistent state."
---

## Phase 1 — Triage

MUST:
- Get deployment details including rollback info: `aws codedeploy get-deployment --deployment-id <deployment-id>`
- Check auto-rollback configuration: `aws codedeploy get-deployment-group --application-name <app> --deployment-group-name <dg> --query 'deploymentGroupInfo.autoRollbackConfiguration'`
- Check if rollback was triggered by alarm: `aws codedeploy get-deployment-group --query 'deploymentGroupInfo.alarmConfiguration'`
- If alarm-triggered: check alarm state: `aws cloudwatch describe-alarms --alarm-names <alarm-name>`

SHOULD:
- Check the rollback deployment status (rollback creates a new deployment)
- Verify the previous revision is still available in S3 or the revision registry
- Check deployment history for the rollback deployment: `aws codedeploy list-deployments --application-name <app> --deployment-group-name <dg> --max-items 5`

MAY:
- Check CloudTrail for StopDeployment or CreateDeployment events
- Review CloudWatch alarm history for false positives
- Check if the rollback deployment itself failed

## Phase 2 — Remediate

MUST:
- If rollback not triggering: verify auto-rollback is enabled in the deployment group
- If alarm not triggering rollback: verify alarm is in ALARM state (not INSUFFICIENT_DATA)
- If rollback deployment failed: manually deploy the last known good revision
- If unwanted rollbacks: review and adjust CloudWatch alarm thresholds

SHOULD:
- For blue/green: verify the original target group still has healthy instances
- Ensure the previous revision artifact is still available in S3
- Review alarm configuration to prevent false-positive rollbacks

MAY:
- Disable auto-rollback temporarily for debugging: update deployment group configuration
- Create a manual deployment with the last known good revision
- Implement canary deployments to catch issues before full rollout

## Common Issues

- symptoms: "Deployment rolled back but rollback also failed"
  diagnosis: "The previous revision has issues or instances are in a bad state."
  resolution: "Manually deploy a known-good revision. Check instance health before redeploying."

- symptoms: "Auto-rollback not triggering on deployment failure"
  diagnosis: "Auto-rollback is not enabled in the deployment group configuration."
  resolution: "Enable auto-rollback: update deployment group with autoRollbackConfiguration enabled for DEPLOYMENT_FAILURE."

- symptoms: "Rollback triggered by CloudWatch alarm but the alarm seems wrong"
  diagnosis: "CloudWatch alarm threshold is too sensitive, causing false-positive rollbacks."
  resolution: "Adjust alarm threshold, evaluation periods, or datapoints to alarm. Consider using composite alarms."

- symptoms: "Blue/green rollback failed — original instances terminated"
  diagnosis: "Original instances were terminated before rollback (termination wait time expired)."
  resolution: "Increase the termination wait time in the blue/green deployment configuration. Default is 0 (immediate termination)."

## Output Format

```yaml
root_cause: "Deployment rollback — <specific_cause>"
evidence:
  - type: deployment_status
    content: "<rollback trigger and status>"
  - type: alarm_state
    content: "<CloudWatch alarm details if applicable>"
severity: HIGH
mitigation:
  immediate: "Resolve the rollback issue and deploy a known-good revision"
  long_term: "Tune alarm thresholds, implement canary deployments, increase blue/green termination wait time"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If rollback not triggering: verify auto-rollback is enabled | GREEN |
| If alarm not triggering rollback: verify alarm is in ALARM state | GREEN |
| If rollback deployment failed: manually deploy the last known good revision | RED |
| If unwanted rollbacks: review and adjust CloudWatch alarm thresholds | YELLOW |
| For blue/green: verify the original target group still has healthy instances | GREEN |
| Ensure the previous revision artifact is still available in S3 | GREEN |
| Review alarm configuration to prevent false-positive rollbacks | YELLOW |
| Disable auto-rollback temporarily for debugging | RED |
| Create a manual deployment with the last known good revision | YELLOW |
| Implement canary deployments | YELLOW |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Rollback configurations reveal deployment strategies and alarm thresholds for production systems

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If auto-rollback was disabled for debugging, re-enable it immediately after investigation
2. If CloudWatch alarm thresholds were adjusted, restore previous thresholds if the change was temporary
3. If a manual deployment was created, verify it matches the last known good revision
4. If blue/green termination wait time was changed, restore the previous value
5. Verify rollback by confirming deployment group auto-rollback configuration matches expected state

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
