---
title: "E2 — WAF Integration"
description: "Diagnose WAF integration issues with Verified Access"
status: active
severity: HIGH
triggers: ["WAF blocking verified access", "web ACL not working", "WAF rules interfering", "rate limiting verified access"]
owner: devops-agent
objective: "Resolve WAF integration issues with Verified Access instances"
context: "WAF can be associated with Verified Access instances for additional security. WAF evaluation happens BEFORE Cedar policy evaluation. WAF blocks appear as connection failures, not policy denials."
---

## Phase 1 — Triage

MUST:
- Check WAF association: `aws wafv2 get-web-acl-for-resource --resource-arn <verified-access-instance-arn>`
- Review WAF logs for blocked requests: `aws logs filter-log-events --log-group-name aws-waf-logs-<name> --filter-pattern '{ $.action = "BLOCK" }' --start-time <epoch-ms>`
- List WAF rules: `aws wafv2 get-web-acl --name <acl-name> --scope REGIONAL --id <acl-id> --query 'WebACL.Rules[*].{Name:Name,Priority:Priority,Action:Action}'`

SHOULD:
- Check if WAF rate limiting is triggering on legitimate traffic
- Verify WAF rules are not blocking Verified Access health checks
- Review WAF managed rule groups for false positives

MAY:
- Enable WAF logging if not configured
- Analyze WAF metrics in CloudWatch

## Phase 2 — Remediate

MUST:
- Identify which WAF rule is blocking legitimate traffic
- Create WAF rule exceptions for false positives
- Ensure WAF rules do not interfere with Verified Access authentication flow

SHOULD:
- Test WAF rules in count mode before switching to block
- Configure WAF logging for ongoing monitoring

MAY:
- Implement WAF rate limiting with appropriate thresholds
- Create custom WAF rules for Verified Access-specific patterns

## Common Issues

- symptoms: "Users get connection errors but Verified Access access logs show no entries"
  diagnosis: "WAF is blocking requests before they reach Verified Access policy evaluation."
  resolution: "Check WAF logs for BLOCK actions. WAF evaluates before Verified Access. Fix WAF rules or add exceptions."

## Safety Ratings

```
safety_ratings:
  - "wafv2 get-web-acl-for-resource: GREEN — read-only association check"
  - "WAF log analysis: GREEN — read-only log review"
  - "WAF rule modifications: YELLOW — changes security filtering rules"
  - "Disassociate WAF from VA: RED — removes security layer"
```

## Escalation Conditions

- WAF is blocking all traffic to Verified Access endpoints
- WAF managed rules causing widespread false positives
- Rate limiting thresholds need adjustment affecting security posture
- WAF and Verified Access interaction causing unexpected behavior

## Data Sensitivity

- WAF association: LOW — reveals security configuration
- WAF logs: HIGH — contain source IPs, request details, and blocking reasons
- WAF rules: MEDIUM — reveals security filtering logic
- Request patterns: MEDIUM — reveals traffic patterns and potential attack vectors

## Prohibited Actions

- NEVER disassociate WAF from Verified Access without security team approval
- NEVER disable WAF rules to troubleshoot without a time-limited plan
- NEVER set WAF default action to ALLOW without explicit security review
- NEVER share WAF logs containing source IPs without authorization

## Phase 3 — Rollback

1. Revert WAF rule changes to previous configuration
2. If WAF was disassociated, re-associate it: `aws wafv2 associate-web-acl --web-acl-arn <acl-arn> --resource-arn <va-instance-arn>`
3. If rules were switched from BLOCK to COUNT, revert to BLOCK after testing
4. Verify WAF is functioning correctly after rollback
5. Document WAF changes and their impact on access patterns

## Output Format

```yaml
root_cause: "waf_integration — <specific_cause>"
evidence:
  - type: waf_details
    content: "<WAF rules, logs, and blocking details>"
severity: HIGH
mitigation:
  immediate: "Fix WAF rule causing false positives"
  long_term: "Implement WAF monitoring and rule testing"
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
