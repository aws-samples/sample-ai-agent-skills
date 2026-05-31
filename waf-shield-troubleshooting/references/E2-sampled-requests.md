---
title: "E2 — Sampled Requests Analysis"
description: "Diagnose issues with WAF sampled requests and use them for troubleshooting"
status: active
severity: MEDIUM
triggers:
  - "sampled requests"
  - "request samples"
  - "WAF request details"
  - "blocked request details"
owner: devops-agent
objective: "Effectively use sampled requests to diagnose WAF rule behavior"
context: "WAF provides sampled requests for each rule in the web ACL. Samples are available for up to 3 hours and include request headers, source IP, URI, HTTP method, country, and the action taken. Sampled requests are limited to approximately 5,000 requests per 3-hour window. They do not include the request body. For full request details, use WAF logging."
---

## Phase 1 — Triage

MUST:
- Get sampled requests for a specific rule: `aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name <metric-name> --scope <scope> --time-window StartTime=<start>,EndTime=<end> --max-items 100`
- Check the time window (samples are only available for the last 3 hours)
- Verify the rule metric name matches the rule you're investigating
- Review the request details: source IP, URI, headers, action, timestamp

SHOULD:
- Compare sampled requests across multiple rules to trace request evaluation
- Check labels applied to sampled requests
- Look for patterns in blocked requests (common IPs, URIs, user agents)

MAY:
- Export sampled requests for offline analysis
- Correlate sampled requests with application logs

## Phase 2 — Remediate

MUST:
- Use sampled request data to identify the specific rule and condition causing blocks
- Verify the source IP, URI, and headers match the expected traffic pattern
- Cross-reference with WAF logs for complete request details

SHOULD:
- Set up WAF logging for persistent request records (sampled requests expire after 3 hours)
- Use sampled requests to validate rule changes before and after modifications
- Document findings from sampled request analysis

MAY:
- Create automated sampled request analysis scripts
- Set up periodic sampled request collection for trend analysis

## Common Issues

- symptoms: "No sampled requests returned for a rule"
  diagnosis: "The rule has not matched any requests in the time window, or the time window is outside the 3-hour limit."
  resolution: "Adjust the time window to the last 3 hours. Verify the rule metric name."

- symptoms: "Sampled requests show unexpected source IPs"
  diagnosis: "Requests are coming through a CDN/proxy. The source IP is the proxy, not the client."
  resolution: "Check X-Forwarded-For header in the sampled request headers for the real client IP."

- symptoms: "Need request body details but sampled requests don't include it"
  diagnosis: "Sampled requests never include the request body — this is by design."
  resolution: "Enable WAF logging for full request details (headers only, body is never logged)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Set up WAF logging | YELLOW | Configuration change; reversible |
| Validate rule changes | GREEN | Read-only analysis |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-sampled-requests` | MEDIUM | Request headers, IPs, URIs |
| WAF logs | HIGH | Full request headers and client IPs |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| WAF logging setup | Disable via `delete-logging-configuration` |
| Rule changes based on analysis | Revert via `update-web-acl` |

## Output Format

```yaml
root_cause: "sampled_requests — <specific_finding>"
evidence:
  - type: sampled_request
    content: "<request details from sample>"
  - type: rule_match
    content: "<rule that matched and action taken>"
  - type: request_pattern
    content: "<common patterns in sampled requests>"
severity: MEDIUM
mitigation:
  immediate: "Use sampled request data to identify and fix the rule issue"
  long_term: "Enable WAF logging for persistent and comprehensive request records"
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
  - "NEVER suggest disabling WAF to fix access issues"
  - "NEVER suggest removing all WAF rules"
  - "NEVER suggest allowing all IPs to bypass rate limiting"
