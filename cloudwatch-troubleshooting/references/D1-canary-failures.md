---
title: "D1 — Canary Failures"
description: "Diagnose Synthetics canary execution failures"
status: active
severity: HIGH
triggers:
  - "canary failed"
  - "canary error"
  - "synthetics failure"
  - "canary not running"
  - "canary status"
owner: devops-agent
objective: "Identify why a Synthetics canary is failing and restore successful execution"
context: "Synthetics canaries run on Lambda functions managed by the service. Failures are caused by script errors, timeout (Lambda 15-min limit), IAM permission issues, target endpoint unavailability, runtime version deprecation, or resource limits. Canary artifacts (screenshots, HAR files, logs) are stored in S3."
---

## Phase 1 — Triage

MUST:
- Check canary status: `aws synthetics describe-canaries --names <canary-name>`
- Get recent run results: `aws synthetics get-canary-runs --name <canary-name> --max-results 5`
- Check the failure reason in the run result
- Verify the canary's IAM execution role has required permissions

SHOULD:
- Check canary artifacts in S3 for screenshots and logs
- Verify the runtime version is not deprecated: check RuntimeVersion in describe-canaries
- Check if the target endpoint is reachable from the canary's network context

MAY:
- Check Lambda function logs for the canary (log group: /aws/lambda/cwsyn-<canary-name>-*)
- Verify S3 bucket for artifacts exists and has correct permissions

## Phase 2 — Remediate

MUST:
- Fix script errors identified in the failure reason
- Update deprecated runtime versions
- Fix IAM execution role permissions

SHOULD:
- Increase timeout if canary is timing out (max 15 minutes due to Lambda limit)
- Update the canary schedule if running too frequently
- Check and fix S3 artifact bucket permissions

MAY:
- Enable active tracing for X-Ray integration
- Set up CloudWatch alarms on canary SuccessPercent metric

## Common Issues

- symptoms: "Canary status is ERROR"
  diagnosis: "Canary script has a runtime error or the execution role lacks permissions."
  resolution: "Check get-canary-runs for the specific error. Fix script or IAM role."

- symptoms: "Canary times out"
  diagnosis: "Target endpoint is slow or unreachable, or script has infinite waits."
  resolution: "Increase timeout (max 840 seconds for API canaries, limited by Lambda). Optimize script."

- symptoms: "Canary was working but started failing after runtime update"
  diagnosis: "Breaking changes in the new Synthetics runtime version."
  resolution: "Check runtime release notes. Pin to a specific runtime version or update the script."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix script errors | GREEN | Code fix, no infrastructure change |
| Update runtime version | YELLOW | May affect canary behavior |
| Fix IAM execution role | YELLOW | IAM changes affect canary capabilities |
| Increase timeout | GREEN | Configuration change |

## Escalation Conditions

- Production canary monitoring broken — escalate to operations team
- Canary script changes needed for target application changes — escalate to application team
- Runtime deprecation affecting multiple canaries — escalate to monitoring team
- IAM role changes for canary execution — escalate to IAM administrator

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws synthetics describe-canaries` | MEDIUM | Exposes canary configuration and target URLs |
| `aws synthetics get-canary-runs` | MEDIUM | Exposes run results and failure details |
| Canary S3 artifacts (screenshots, HAR) | HIGH | May contain application UI and network data |
| Lambda logs for canary | MEDIUM | May contain application response data |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** store credentials in canary scripts; use Secrets Manager or environment variables

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Updated canary script | Revert to previous script version |
| Updated runtime version | Revert to previous runtime (if still supported) |
| Fixed IAM role | Revert IAM role changes |
| Increased timeout | Restore previous timeout value |

## Output Format

```yaml
root_cause: "canary_failure — <specific_cause>"
evidence:
  - type: canary_config
    content: "<canary configuration>"
  - type: run_results
    content: "<recent canary run failures>"
severity: HIGH
mitigation:
  immediate: "Fix script, IAM role, or runtime version"
  long_term: "Monitor SuccessPercent metric and set up failure alarms"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
