---
name: waf-shield-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS WAF and AWS Shield problems
  by analyzing web ACL configurations, rule evaluation, managed rule groups,
  IP sets, rate-based rules, logging, Shield Advanced protections, and following
  structured runbooks. Activate when: false positives blocking legitimate traffic,
  false negatives allowing malicious traffic, rule priority issues, rate-based
  rule problems, managed rule group conflicts, IP blocking/allowing failures,
  geo-restriction issues, body inspection limits, regex pattern problems,
  WAF logging issues, sampled request analysis, CloudWatch metric anomalies,
  ALB/CloudFront/API Gateway integration problems, DDoS detection, Shield
  Advanced configuration, SRT engagement, cost protection claims, Bot Control
  issues, Account Takeover Prevention, custom response bodies, rate limiting
  responses, or the user says something is wrong with WAF or Shield without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with WAFv2, Shield, CloudWatch, CloudTrail,
  Kinesis Firehose, S3, ALB, CloudFront, API Gateway, and Route 53 permissions.
---

# WAF & Shield Diagnostics

## When to use

Any WAF or Shield investigation where the console alone is insufficient — rule evaluation analysis, false positive/negative debugging, managed rule group troubleshooting, DDoS protection verification, logging configuration, integration issues, or Bot Control and ATP tuning.

## Investigation workflow

### Step 1 — Collect and triage

```
aws wafv2 list-web-acls --scope REGIONAL
aws wafv2 list-web-acls --scope CLOUDFRONT --region us-east-1
aws wafv2 get-web-acl --name <acl-name> --scope <REGIONAL|CLOUDFRONT> --id <acl-id>
aws wafv2 get-web-acl-for-resource --resource-arn <resource-arn>
aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 --metric-name AllowedRequests --dimensions Name=WebACL,Value=<acl-name> Name=Rule,Value=ALL Name=Region,Value=<region> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 --metric-name BlockedRequests --dimensions Name=WebACL,Value=<acl-name> Name=Rule,Value=ALL Name=Region,Value=<region> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Rule deep dive

```
aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[*].{Name:Name,Priority:Priority,Action:Action,OverrideAction:OverrideAction}'
version: "1.0.0"
last_updated: "2025-04-12"
aws wafv2 get-rule-group --name <group-name> --scope <scope> --id <group-id>
aws wafv2 list-managed-rule-sets --scope <scope>
aws wafv2 describe-managed-rule-group --vendor-name AWS --name <rule-group-name> --scope <scope>
aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> --rule-metric-name <metric-name> --scope <scope> --time-window StartTime=<start>,EndTime=<end> --max-items 100
aws wafv2 get-rate-based-statement-managed-keys --scope <scope> --web-acl-name <acl-name> --web-acl-id <acl-id> --rule-name <rule-name>
```

### Step 3 — Detailed investigation

```
aws wafv2 get-logging-configuration --resource-arn <web-acl-arn>
aws wafv2 list-ip-sets --scope <scope>
aws wafv2 get-ip-set --name <ip-set-name> --scope <scope> --id <ip-set-id>
aws wafv2 list-regex-pattern-sets --scope <scope>
aws wafv2 get-regex-pattern-set --name <set-name> --scope <scope> --id <set-id>
aws shield describe-subscription
aws shield list-protections
aws shield describe-protection --resource-arn <resource-arn>
aws shield describe-attack --attack-id <attack-id>
aws shield list-attacks --start-time FromInclusive=<start>,ToExclusive=<end>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wafv2.amazonaws.com
```

Read `references/waf-shield-guardrails.md` before concluding on any WAF or Shield issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `wafv2 get-web-acl` | Full web ACL details including all rules |
| `wafv2 get-web-acl-for-resource` | Find which web ACL protects a resource |
| `wafv2 get-sampled-requests` | Inspect recent requests matched by rules |
| `wafv2 get-rule-group` | Custom rule group details |
| `wafv2 describe-managed-rule-group` | Managed rule group rules and labels |
| `wafv2 get-rate-based-statement-managed-keys` | IPs currently rate-limited |
| `wafv2 get-logging-configuration` | WAF logging destination |
| `wafv2 get-ip-set` | IP set addresses |
| `wafv2 get-regex-pattern-set` | Regex patterns in a set |
| `shield describe-subscription` | Shield Advanced subscription status |
| `shield list-protections` | Resources protected by Shield Advanced |
| `shield describe-attack` | DDoS attack details |

## Gotchas: WAF & Shield

- Rule evaluation order: rules are evaluated by priority number, lowest number first. If a request matches a rule with priority 1 (Block), it is blocked before priority 2 (Allow) is ever evaluated. Priority numbers must be unique within a web ACL.
- WAF Capacity Units (WCU): each web ACL has a 1,500 WCU limit. Each rule consumes WCUs based on complexity. Text transformations, regex patterns, and size constraint statements cost more WCUs. Managed rule groups publish their WCU requirements. Exceeding the limit prevents adding more rules.
- Rate-based rule minimum: the minimum rate limit is 100 requests per 5-minute window. You cannot set a threshold lower than 100. Rate evaluation uses a sliding window, not fixed 5-minute blocks. IPs are blocked once they exceed the threshold and unblocked when they drop below.
- Managed rule group versioning: AWS managed rule groups are versioned. By default, the web ACL uses the default version set by AWS. You can pin to a specific version. AWS may update the default version, which can change blocking behavior. Always test version changes in Count mode first.
- Shield Standard vs Advanced: Shield Standard is free and automatically protects all AWS resources against common L3/L4 DDoS attacks. Shield Advanced ($3,000/month per organization) adds L7 protection, DDoS cost protection, 24/7 SRT access, advanced metrics, and WAF fee waiver for resources protected by Shield Advanced.
- Shield Response Team (SRT) requirements: SRT engagement requires Shield Advanced subscription AND an IAM role granting SRT access to your WAF and Shield resources. Without the IAM role, SRT cannot assist. Create the role using `aws shield create-subscription` and `aws shield associate-drt-role`.
- WAF logging: logs must go to Amazon Kinesis Data Firehose, S3 bucket, or CloudWatch Logs log group. Firehose delivery stream name MUST start with `aws-waf-logs-`. S3 bucket name MUST start with `aws-waf-logs-`. CloudWatch log group MUST start with `aws-waf-logs-`. Logs include full request headers but NOT the request body.
- Regex pattern set limits: maximum 10 regex patterns per regex pattern set. Each pattern can be up to 200 characters. Regex uses Java-compatible syntax (PCRE-like). Complex regex patterns consume more WCUs.
- IP set limits: maximum 10,000 IP addresses per IP set. Supports IPv4 and IPv6 CIDR notation. You must specify the IP version when creating the set. Use /32 for single IPv4 addresses and /128 for single IPv6 addresses.
- Scope — REGIONAL vs CLOUDFRONT: REGIONAL scope protects ALB, API Gateway, AppSync, Cognito, App Runner, and Verified Access. CLOUDFRONT scope protects CloudFront distributions and MUST be created in us-east-1. A web ACL in REGIONAL scope cannot be attached to CloudFront and vice versa.
- Rule action override in managed groups: when adding a managed rule group to a web ACL, you can override individual rule actions to Count instead of Block. This lets you test rules before enforcing them. Use `OverrideAction` at the group level (None or Count) and `RuleActionOverrides` for individual rules.
- Custom response bodies: you can define custom response bodies (up to 4 KB each, max 50 per web ACL) in JSON, HTML, or plain text. Custom responses can include custom headers. Response status codes can be 200-599. Custom responses only apply to Block actions.
- Label matching across rules: rules can add labels to matching requests. Subsequent rules (higher priority number) can match on those labels using label match statements. Labels are scoped to the web ACL evaluation context and do not persist beyond the evaluation. Labels enable multi-stage rule logic.
- Bot Control and ATP costs: Bot Control common level is included with WAF. Bot Control targeted level costs $10 per million requests. Account Takeover Prevention (ATP) costs $10 per thousand login attempts analyzed. These are in addition to standard WAF charges. Enable in Count mode first to estimate costs.

## Anti-hallucination rules

1. Always cite specific web ACL configurations, rule priorities, sampled requests, or API responses as evidence.
2. Rule evaluation order is by priority number, lowest first. Never claim rules are evaluated alphabetically, by creation date, or by any other ordering.
3. Rate-based rules have a minimum threshold of 100 requests per 5-minute window. Never claim you can set a lower threshold.
4. Shield Standard does NOT provide L7 DDoS protection, DDoS cost protection, or SRT access. Never claim Shield Standard includes these features.
5. WAF logging destination names MUST start with `aws-waf-logs-`. Never omit this prefix requirement.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 32 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Rule Evaluation | A1-A4 | False positives, false negatives, rule priority, rate-based rules |
| B — Managed Rules | B1-B3 | AWS managed rules, marketplace rules, rule group versioning |
| C — IP & Geo | C1-C3 | IP blocking/allowing, geo-restriction, IP set management |
| D — Request Inspection | D1-D3 | Body inspection limits, regex issues, size constraints |
| E — Logging & Monitoring | E1-E3 | WAF logging setup, sampled requests, CloudWatch metrics |
| F — Integration | F1-F3 | ALB integration, CloudFront integration, API Gateway integration |
| G — Shield | G1-G4 | DDoS detection, Shield Advanced, SRT engagement, cost protection |
| H — Bot Control | H1-H2 | Bot Control rules, Account Takeover Prevention |
| I — Custom Responses | I1-I2 | Custom response bodies, rate limiting responses |
| Z — Catch-All | Z1 | General troubleshooting |
