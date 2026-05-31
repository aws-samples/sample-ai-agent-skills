---
title: "E2 — WAF Integration"
description: "Diagnose and resolve ELB and AWS WAF integration issues"
status: active
severity: HIGH
triggers:
  - "WAF blocking"
  - "WAF rule"
  - "WAF association"
  - "web ACL"
  - "WAF 403"
owner: devops-agent
objective: "Resolve WAF integration issues with ELB and ensure proper traffic filtering"
context: "AWS WAF can be associated with ALBs to filter HTTP/HTTPS traffic based on rules. WAF web ACLs contain rules that inspect requests and take actions (allow, block, count). Issues arise from incorrect rule configuration, false positives blocking legitimate traffic, WAF not associated with the ALB, or rate-based rule misconfiguration. WAF is only supported on ALB, not NLB or CLB."
---

## Phase 1 — Triage

MUST:
- Check if a WAF web ACL is associated with the ALB: `aws wafv2 get-web-acl-for-resource --resource-arn <alb-arn>`
- List WAF rules in the web ACL: `aws wafv2 get-web-acl --name <acl-name> --scope REGIONAL --id <acl-id>`
- Check WAF sampled requests for blocked traffic: `aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name <rule-metric> --scope REGIONAL --time-window StartTime=<start>,EndTime=<end> --max-items 10`
- Review CloudWatch WAF metrics: `aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 --metric-name BlockedRequests --dimensions Name=WebACL,Value=<acl-name> Name=Region,Value=<region> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Check the default action of the web ACL (allow or block)

SHOULD:
- Review WAF logging for detailed request information: check the WAF log destination (S3, CloudWatch Logs, or Kinesis)
- Identify which specific rule is blocking traffic from sampled requests
- Check if managed rule groups have known false positive patterns

MAY:
- Test specific requests against WAF rules using the WAF testing tools
- Review rate-based rule thresholds and current request rates
- Check if IP set or regex pattern rules are too broad

## Phase 2 — Remediate

MUST:
- If WAF not associated: associate the web ACL: `aws wafv2 associate-web-acl --web-acl-arn <acl-arn> --resource-arn <alb-arn>`
- If legitimate traffic blocked: add an exception rule or adjust the blocking rule
- If false positive from managed rules: add a scope-down statement or label-based exception
- If WAF association fails: verify the web ACL is in the same region as the ALB

SHOULD:
- Set blocking rules to COUNT mode first to evaluate impact before switching to BLOCK
- Enable WAF logging for detailed request analysis: `aws wafv2 put-logging-configuration --logging-configuration ResourceArn=<acl-arn>,LogDestinationConfigs=<log-dest-arn>`
- Review and tune rate-based rule thresholds based on normal traffic patterns

MAY:
- Implement custom WAF rules for application-specific protection
- Set up CloudWatch alarms for unusual block rates
- Document the WAF rule set and exception rationale

## Common Issues

- symptoms: "All requests return 403 after WAF association"
  diagnosis: "Web ACL default action is set to BLOCK with no allow rules."
  resolution: "Change the default action to ALLOW, or add rules that allow legitimate traffic."

- symptoms: "Specific API endpoints blocked by WAF"
  diagnosis: "Managed rule group (e.g., SQL injection, XSS) matching on request body or parameters."
  resolution: "Add a scope-down statement to exclude the specific endpoint, or use a label-based exception."

- symptoms: "WAF association fails with 'WAFNonexistentItemException'"
  diagnosis: "Web ACL ID or ARN is incorrect, or the ACL is in a different region."
  resolution: "Verify the web ACL exists in the same region as the ALB."

- symptoms: "Rate-based rule blocking legitimate users"
  diagnosis: "Rate limit threshold is too low for normal traffic patterns."
  resolution: "Increase the rate limit threshold. Consider using IP-based rate limiting with scope-down statements."

## Output Format

```yaml
root_cause: "WAF integration issue — <specific_cause>"
evidence:
  - type: waf_config
    content: "<web ACL rules and default action>"
  - type: blocked_requests
    content: "<sampled blocked request details>"
severity: HIGH
mitigation:
  immediate: "Fix WAF rules or association configuration"
  long_term: "Implement WAF logging, monitoring, and rule tuning process"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get/list/describe): GREEN — read-only"
  - "Phase 2 WAF rule changes: RED — directly affects traffic filtering"
  - "Phase 2 WAF association/disassociation: RED — enables/disables security controls"

## Escalation Conditions

escalation_conditions:
  - "WAF is blocking legitimate production traffic"
  - "WAF rule changes could expose the application to attacks"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "WAF changes require security team approval"
  - "Issue involves compliance-required WAF rules (PCI DSS, etc.)"

## Data Sensitivity

data_sensitivity:
  - command: "get-sampled-requests"
    sensitivity: HIGH
    contains: "Client IPs, request headers, and body content"
  - command: "WAF logs"
    sensitivity: HIGH
    contains: "Detailed request information including potential attack payloads"
  - command: "get-web-acl"
    sensitivity: MEDIUM
    contains: "Security rule configuration"

## Prohibited Actions

prohibited_actions:
  - "NEVER disassociate WAF from a production ALB without security team approval"
  - "NEVER change WAF default action to ALLOW without reviewing all rules"
  - "NEVER delete WAF rules without understanding their security purpose"
  - "NEVER disable rate-based rules during an active attack"
  - "NEVER share WAF sampled request data containing client information"

## Phase 3 — Rollback

- Before modifying WAF rules, export the current web ACL configuration
- If rule changes cause legitimate traffic to be blocked, revert the rule changes
- If WAF was disassociated, re-associate it immediately
- If rate limits were changed, restore the previous thresholds
- Monitor blocked request rates and application availability after any changes
