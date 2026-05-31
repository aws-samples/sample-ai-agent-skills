---
title: "G2 — Rewrite and Redirect Rule Issues"
description: "Diagnose rewrite and redirect rules not matching or misconfigured"
status: active
severity: MEDIUM
triggers:
  - "rewrite not working"
  - "redirect not working"
  - "404 on refresh"
  - "SPA routing broken"
  - "URL rewrite"
owner: devops-agent
objective: "Identify and resolve rewrite and redirect rule issues"
context: "Rewrite/redirect issues include: SPA catch-all rule missing (404 on refresh), rule order wrong, regex pattern invalid, trailing slash mismatch, or conflicting rules."
---

## Phase 1 — Triage

MUST:
- Check custom rules: `aws amplify get-app --app-id <app-id> --query 'app.customRules'`
- Verify rule order (first match wins)
- Check if SPA catch-all rule exists for single-page apps
- Test specific URLs against rules

SHOULD:
- Verify regex patterns are valid JavaScript regex
- Check for trailing slash issues
- Verify status codes (200 for rewrite, 301/302 for redirect)

MAY:
- Test rules with Amplify's rule testing tool in console
- Check if rules conflict with file paths in the build output

## Phase 2 — Remediate

MUST:
- For SPAs: add catch-all rewrite rule: `</^[^.]+$|\.(?!(css|gif|ico|jpg|js|png|txt|svg|woff|woff2|ttf|map|json)$)([^.]+$)/> /index.html 200`
- Fix rule order — most specific rules first, catch-all last
- Correct regex patterns

SHOULD:
- Update rules: `aws amplify update-app --app-id <app-id> --custom-rules '[{"source":"/<*>","target":"/index.html","status":"200"}]'`
- Add trailing slash redirect if needed

MAY:
- Implement proxy rewrites for API calls
- Set up country-based redirects

## Safety Ratings

```
safety_ratings:
  - "get-app (customRules): GREEN — read-only rule check"
  - "test URLs against rules: GREEN — read-only verification"
  - "update-app (custom-rules): YELLOW — changes URL routing behavior"
  - "add SPA catch-all rule: YELLOW — changes how 404s are handled"
```

## Escalation Conditions

- Rewrite rules causing 404 errors on production SPA
- Redirect rules sending users to wrong URLs
- Rule conflicts causing unpredictable routing behavior
- Proxy rewrite rules exposing internal API endpoints

## Data Sensitivity

- Custom rules: LOW — URL routing patterns
- Failing URLs: LOW — URL paths
- Proxy rewrite targets: MEDIUM — may reveal internal API endpoints

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER create proxy rewrite rules that expose internal services without authentication
- NEVER remove the SPA catch-all rule without understanding the impact on client-side routing

## Phase 3 — Rollback

1. If custom rules were changed incorrectly, restore previous rules: `aws amplify update-app --app-id <id> --custom-rules '<previous-rules>'`
2. If the SPA catch-all rule was removed, re-add it immediately
3. If proxy rewrite rules were added incorrectly, remove them
4. Verify all application routes work correctly after rollback
5. Test with both direct URL access and client-side navigation

## Output Format

```yaml
root_cause: "rewrite_rule_issue — <specific_cause>"
evidence:
  - type: custom_rules
    content: "<configured rules>"
  - type: failing_url
    content: "<URL that doesn't match>"
severity: MEDIUM
mitigation:
  immediate: "Fix rule order or add missing catch-all rule"
  long_term: "Document and test all rewrite/redirect rules"
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
