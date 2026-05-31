---
title: "D3 — Canary Script Errors"
description: "Diagnose Synthetics canary script runtime errors"
status: active
severity: MEDIUM
triggers:
  - "canary script"
  - "canary runtime"
  - "canary Node.js"
  - "canary Python"
  - "canary selenium"
  - "canary puppeteer"
owner: devops-agent
objective: "Identify and fix script errors in Synthetics canary code"
context: "Canary scripts run in Node.js (Puppeteer/Selenium) or Python (Selenium) runtimes. Common script errors include incorrect selectors, timing issues (element not loaded), assertion failures, import errors from wrong runtime version, and missing environment variables."
---

## Phase 1 — Triage

MUST:
- Get canary run failure details: `aws synthetics get-canary-runs --name <canary-name> --max-results 5`
- Check the failure reason and error message in the run result
- Review canary script code for syntax errors
- Verify the runtime version matches the script's API usage

SHOULD:
- Check canary Lambda logs: log group `/aws/lambda/cwsyn-<canary-name>-*`
- Review screenshots in S3 artifacts to see what the page looked like at failure
- Check HAR file for network request failures

MAY:
- Test the script locally using the Synthetics library
- Check if the target page structure changed (broken selectors)

## Phase 2 — Remediate

MUST:
- Fix script syntax and logic errors
- Update selectors if the target page structure changed
- Match runtime version to the script's API requirements

SHOULD:
- Add explicit waits for elements instead of fixed timeouts
- Add error handling and meaningful assertion messages
- Use the latest stable runtime version

MAY:
- Implement retry logic for flaky elements
- Add custom metrics from the canary script for detailed monitoring

## Common Issues

- symptoms: "TimeoutError: waiting for selector"
  diagnosis: "Page element not found within the wait timeout. Page structure may have changed."
  resolution: "Update the CSS/XPath selector. Increase wait timeout. Check screenshots for page state."

- symptoms: "ReferenceError or ImportError in canary"
  diagnosis: "Script uses APIs not available in the configured runtime version."
  resolution: "Check runtime version compatibility. Update script or runtime version."

- symptoms: "Canary passes locally but fails in AWS"
  diagnosis: "Environment differences: missing environment variables, network restrictions, or different browser version."
  resolution: "Check environment variables in canary config. Verify network access. Match runtime version."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix script syntax and logic | GREEN | Code fix, no infrastructure change |
| Update selectors | GREEN | Script update |
| Update runtime version | YELLOW | May affect canary behavior |
| Add error handling | GREEN | Script improvement |

## Escalation Conditions

- Canary script broken due to target application changes — escalate to application team
- Runtime version incompatibility — escalate to monitoring team
- Persistent script failures affecting SLA monitoring — escalate to operations team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws synthetics get-canary-runs` | MEDIUM | Exposes error messages and failure details |
| Canary Lambda logs | MEDIUM | May contain application response data |
| S3 artifacts (screenshots, HAR) | HIGH | May contain application UI and network data |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** hardcode credentials in canary scripts

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Updated canary script | Revert to previous script version |
| Updated runtime version | Revert to previous runtime |

## Output Format

```yaml
root_cause: "canary_script — <specific_cause>"
evidence:
  - type: run_failure
    content: "<error message from canary run>"
  - type: script_analysis
    content: "<identified script issue>"
severity: MEDIUM
mitigation:
  immediate: "Fix script errors, selectors, or runtime version"
  long_term: "Add robust waits, error handling, and screenshot analysis"
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
