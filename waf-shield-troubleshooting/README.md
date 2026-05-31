# WAF & Shield Diagnostics Skill

Agent skill for investigating and troubleshooting AWS WAF and AWS Shield problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS WAF and Shield when the console alone isn't enough — rule evaluation analysis, false positive/negative debugging, managed rule group configuration, IP and geo-restriction, request inspection limits, logging and monitoring, resource integration, DDoS protection, Bot Control, Account Takeover Prevention, and custom response configuration.

### Activate When

- False positives blocking legitimate traffic
- False negatives allowing malicious requests through
- Rule priority or evaluation order confusion
- Rate-based rules not triggering or over-blocking
- AWS managed rule group unexpected blocks
- Marketplace rule group issues
- Rule group version changes causing problems
- IP blocking or allowing not working
- Geo-restriction failures
- IP set management issues (size limits, CIDR format)
- Request body not being inspected
- Regex pattern matching failures
- Size constraint statement issues
- WAF logging not working or missing logs
- Sampled requests analysis needed
- CloudWatch WAF metrics anomalies
- ALB + WAF integration problems
- CloudFront + WAF integration problems
- API Gateway + WAF integration problems
- DDoS attack detection and response
- Shield Advanced configuration
- Shield Response Team engagement
- DDoS cost protection claims
- Bot Control rule tuning
- Account Takeover Prevention setup
- Custom response body configuration
- Rate limiting response customization

---

## Skill Structure

```
waf-shield-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-false-positives.md
    ├── A2-false-negatives.md
    ├── A3-rule-priority.md
    ├── A4-rate-based-rules.md
    ├── B1-aws-managed-rules.md
    ├── B2-marketplace-rules.md
    ├── B3-rule-group-versioning.md
    ├── C1-ip-blocking.md
    ├── C2-geo-restriction.md
    ├── C3-ip-set-management.md
    ├── D1-body-inspection-limits.md
    ├── D2-regex-issues.md
    ├── D3-size-constraints.md
    ├── E1-waf-logging.md
    ├── E2-sampled-requests.md
    ├── E3-cloudwatch-metrics.md
    ├── F1-alb-integration.md
    ├── F2-cloudfront-integration.md
    ├── F3-api-gateway-integration.md
    ├── G1-ddos-detection.md
    ├── G2-shield-advanced.md
    ├── G3-srt-engagement.md
    ├── G4-cost-protection.md
    ├── H1-bot-control.md
    ├── H2-account-takeover-prevention.md
    ├── I1-custom-response-bodies.md
    ├── I2-rate-limiting-responses.md
    ├── Z1-general-troubleshooting.md
    ├── waf-shield-guardrails.md
    └── waf-shield-hallucination-patterns.yaml
```

---

## Runbook Library (32 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Rule Evaluation** | A1–A4 | False positives, false negatives, rule priority, rate-based rules |
| **B — Managed Rules** | B1–B3 | AWS managed rules, marketplace rules, rule group versioning |
| **C — IP & Geo** | C1–C3 | IP blocking/allowing, geo-restriction, IP set management |
| **D — Request Inspection** | D1–D3 | Body inspection limits, regex issues, size constraints |
| **E — Logging & Monitoring** | E1–E3 | WAF logging setup, sampled requests, CloudWatch metrics |
| **F — Integration** | F1–F3 | ALB integration, CloudFront integration, API Gateway integration |
| **G — Shield** | G1–G4 | DDoS detection, Shield Advanced, SRT engagement, cost protection |
| **H — Bot Control** | H1–H2 | Bot Control rules, Account Takeover Prevention |
| **I — Custom Responses** | I1–I2 | Custom response bodies, rate limiting responses |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/waf-shield-guardrails.md` covering rule evaluation order (lowest priority number first), WCU limits (1,500 per web ACL), rate-based minimum threshold (100 requests/5 min), WAF logging prefix requirement (`aws-waf-logs-`), Shield Standard vs Advanced capabilities, body inspection limits (8KB/16KB/32KB/64KB), IP set size limits (10,000 per set), scope REGIONAL vs CLOUDFRONT, managed rule group override actions, label namespace scoping, custom response body limits (4KB, max 50), and Shield Advanced cost protection requiring Route 53 health check.

---

## Investigation Workflow

1. **Triage** — Collect web ACL config, check CloudWatch blocked/allowed metrics, identify scope
2. **Rule Deep Dive** — Examine rule priorities, managed rule groups, sampled requests, rate-based keys
3. **Detailed** — Inspect logging config, IP sets, regex patterns, Shield protections, CloudTrail events

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `wafv2:*`, `shield:*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `firehose:DescribeDeliveryStream`, `elasticloadbalancing:DescribeLoadBalancers`, `cloudfront:GetDistribution`, `apigateway:GET`
- WAF logging enabled (recommended)
- CloudWatch metrics enabled for the web ACL

---

## Usage Examples

```
# List web ACLs in a region
aws wafv2 list-web-acls --scope REGIONAL --region us-east-1

# Get web ACL details
aws wafv2 get-web-acl --name my-web-acl --scope REGIONAL --id <acl-id>

# Check blocked requests
aws cloudwatch get-metric-statistics --namespace AWS/WAFV2 \
  --metric-name BlockedRequests --dimensions Name=WebACL,Value=my-web-acl Name=Rule,Value=ALL Name=Region,Value=us-east-1 \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 3600 --statistics Sum

# Get sampled requests for a rule
aws wafv2 get-sampled-requests --web-acl-arn <acl-arn> \
  --rule-metric-name my-rule --scope REGIONAL \
  --time-window StartTime=2024-01-01T00:00:00Z,EndTime=2024-01-01T01:00:00Z --max-items 100

# Check Shield Advanced subscription
aws shield describe-subscription
```

---

## License

MIT-0
