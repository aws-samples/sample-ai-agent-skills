---
title: "E1 — Browser Policy Issues"
description: "Diagnose WorkSpaces Web browser policy configuration issues"
status: active
severity: MEDIUM
triggers: ["browser policy error", "clipboard not working", "file transfer blocked", "printing disabled", "policy JSON error"]
owner: devops-agent
objective: "Resolve browser policy configuration issues affecting session functionality"
context: "Browser policies control session capabilities like clipboard, file transfer, printing, and URL filtering. Policies use Chrome enterprise policy JSON format. Not all Chrome policies are supported."
---

## Phase 1 — Triage

MUST:
- Get current browser settings: `aws workspaces-web get-browser-settings --browser-settings-arn <arn>`
- Verify the browser policy JSON is valid
- Check which portal the browser settings are associated with

SHOULD:
- Identify the specific policy causing the issue (clipboard, file transfer, printing, etc.)
- Compare current policy against the desired configuration
- Check if the policy is a supported Chrome enterprise policy for WorkSpaces Web

MAY:
- Review CloudTrail for recent UpdateBrowserSettings events
- Test with a minimal browser policy to isolate the issue

## Phase 2 — Remediate

MUST:
- Fix browser policy JSON syntax errors
- Update browser settings with corrected policy: `aws workspaces-web update-browser-settings --browser-settings-arn <arn> --browser-policy '{"chromePolicies":{"URLBlocklist":["*://blocked-site.com/*"]}}'`
- Verify the policy takes effect in new sessions (existing sessions may need to be restarted)

SHOULD:
- Use only supported Chrome enterprise policies for WorkSpaces Web
- Test policy changes with a single user before applying broadly
- Document the policy configuration and its business justification

MAY:
- Implement URL allowlisting/blocklisting for content filtering
- Configure extension policies for managed browser extensions
- Set up different browser settings for different portals (different user groups)

## Safety Ratings

```
safety_ratings:
  - "get-browser-settings: GREEN — read-only policy retrieval"
  - "update-browser-settings: YELLOW — modifies browser capabilities for all sessions"
  - "create-browser-settings: YELLOW — creates new policy resource"
  - "associate-browser-settings: YELLOW — changes policy for portal sessions"
```

## Escalation Conditions

- Browser policy changes cause sessions to fail to start
- Required Chrome policy is not supported by WorkSpaces Web
- Policy conflicts between security requirements and user functionality needs
- Extension installation policy does not work as expected

## Data Sensitivity

- `aws workspaces-web get-browser-settings`: MEDIUM — reveals browser policy restrictions
- Browser policy JSON: MEDIUM — reveals security controls and URL filtering rules
- Extension policies: LOW — reveals allowed/blocked extensions

## Prohibited Actions

- NEVER disable all security policies for troubleshooting in production
- NEVER enable clipboard and file transfer without security team approval
- NEVER configure policies that bypass data loss prevention requirements
- NEVER apply untested policy changes to production portals

## Phase 3 — Rollback

1. Revert browser settings to previous policy: `aws workspaces-web update-browser-settings --browser-settings-arn <arn> --browser-policy '<previous-policy-json>'`
2. If browser settings were replaced, re-associate the previous browser settings with the portal
3. Verify new sessions use the reverted policy
4. Notify users if session capabilities changed during troubleshooting
5. Document the policy change attempt and rollback reason

## Common Issues

- symptoms: "UpdateBrowserSettings returns ValidationException"
  diagnosis: "Browser policy JSON is malformed or contains unsupported Chrome policies."
  resolution: "Validate JSON syntax and use only supported Chrome enterprise policies."

## Output Format

```yaml
root_cause: "browser_policy — <specific_cause>"
evidence:
  - type: policy_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix browser policy configuration"
  long_term: "Document standard policies and implement policy testing procedures"
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
