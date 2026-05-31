---
title: "B2 — JavaScript Snippet Issues"
description: "Diagnose JavaScript snippet installation and loading problems"
status: active
severity: HIGH
triggers: ["snippet not working", "JavaScript error", "snippet not loading", "RUM client error"]
owner: devops-agent
objective: "Resolve JavaScript snippet installation and loading issues"
context: "The RUM JavaScript snippet must be embedded in the application HTML and loaded before other scripts. It requires correct app monitor ID and Cognito configuration."
---

## Phase 1 — Triage

MUST:
- Verify the snippet is present in the application HTML source
- Check the snippet configuration: app monitor ID, region, Cognito Identity pool ID
- Verify the snippet loads before other JavaScript files

SHOULD:
- Check browser developer console for JavaScript errors related to RUM
- Verify the snippet version is current
- Check if Content Security Policy blocks the snippet or its network requests

MAY:
- Test the snippet in an isolated HTML page to rule out application conflicts
- Check network tab in browser dev tools for failed requests to RUM or Cognito endpoints
- Verify no ad blockers or browser extensions are blocking RUM

## Phase 2 — Remediate

MUST:
- Install the correct snippet with valid app monitor ID and region
- Ensure the snippet is in the `<head>` section before other scripts
- Fix CSP headers to allow RUM script and endpoint connections

SHOULD:
- Use the latest version of the RUM web client
- Configure the snippet with correct Cognito Identity pool ID
- Test in multiple browsers to verify compatibility

MAY:
- Implement error handling around the snippet initialization
- Add the snippet via a tag manager if direct HTML modification is not possible

## Common Issues

- symptoms: "Snippet installed but browser console shows 'Access to fetch has been blocked by CORS policy'"
  diagnosis: "Content Security Policy or CORS configuration is blocking RUM network requests."
  resolution: "Update CSP to allow connect-src for the RUM endpoint (dataplane.rum.<region>.amazonaws.com) and Cognito endpoint (cognito-identity.<region>.amazonaws.com)."

## Safety Ratings

```
safety_ratings:
  - "HTML source inspection: GREEN — read-only page source review"
  - "Browser console review: GREEN — read-only error log review"
  - "CSP header modification: YELLOW — changes application security policy"
  - "Snippet update: YELLOW — modifies application JavaScript"
```

## Escalation Conditions

- Snippet causes JavaScript errors that affect application functionality
- CSP changes required but security team approval needed
- Snippet works in some browsers but not others
- Snippet conflicts with other JavaScript libraries in the application

## Data Sensitivity

- Snippet configuration: MEDIUM — contains app monitor ID and Cognito pool ID
- Browser console logs: MEDIUM — may contain user session information
- Network requests: MEDIUM — reveal RUM data being sent

## Prohibited Actions

- NEVER disable CSP entirely to fix snippet loading issues
- NEVER modify the snippet to bypass security controls
- NEVER deploy snippet changes to production without testing in staging
- NEVER include sensitive data in custom snippet configuration

## Phase 3 — Rollback

1. If the snippet was updated incorrectly, revert to the previous version
2. If CSP headers were modified, restore original CSP policy
3. If the snippet causes application errors, remove it temporarily and investigate
4. Verify the application functions correctly after snippet changes
5. Document the snippet issue and resolution

## Output Format

```yaml
root_cause: "javascript_snippet — <specific_cause>"
evidence:
  - type: snippet_issue
    content: "<snippet configuration and error details>"
severity: HIGH
mitigation:
  immediate: "Fix snippet installation or CSP configuration"
  long_term: "Implement snippet health monitoring and automated testing"
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
