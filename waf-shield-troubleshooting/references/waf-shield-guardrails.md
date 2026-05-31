# AWS WAF & Shield Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any WAF or Shield issue.

## Guardrail 1: Rule Evaluation Order — Lowest Priority Number First
WAF rules are evaluated by priority number in ascending order. Priority 0 is evaluated first, then 1, 2, etc. If a request matches a rule with a terminating action (Block or Allow), evaluation stops immediately. Count actions do not terminate evaluation — the request continues to the next rule. Priority numbers must be unique within a web ACL. Never assume rules are evaluated alphabetically or by creation date.

## Guardrail 2: WCU Limits — 1,500 Per Web ACL
Each web ACL has a hard limit of 1,500 Web ACL Capacity Units (WCUs). Each rule type consumes different WCU amounts: simple match = 1 WCU, regex match = 25 WCUs, size constraint = 1 WCU, rate-based = 2 WCUs, managed rule groups vary (check with `describe-managed-rule-group`). Text transformations add 10 WCUs each beyond the first. You can request a limit increase through AWS Support but the default is 1,500.

## Guardrail 3: Rate-Based Rule Minimum Threshold — 100 Requests per 5 Minutes
The minimum rate limit for rate-based rules is 100 requests per 5-minute evaluation window. You cannot configure a threshold lower than 100. The evaluation uses a sliding window, not fixed 5-minute blocks. Once an IP exceeds the threshold, it is blocked until the request rate drops below the threshold. Rate-based rules can use custom keys (IP, forwarded IP, header, query string, etc.) beyond just source IP.

## Guardrail 4: WAF Logging Prefix Requirement — aws-waf-logs-
WAF log destinations MUST have names starting with `aws-waf-logs-`. For Kinesis Data Firehose, the delivery stream name must start with `aws-waf-logs-`. For S3, the bucket name must start with `aws-waf-logs-`. For CloudWatch Logs, the log group name must start with `aws-waf-logs-`. Logs include request headers, rule matches, and labels but do NOT include the request body. Log redaction can be configured for specific fields.

## Guardrail 5: Shield Standard vs Shield Advanced
Shield Standard is free and automatic for all AWS accounts. It protects against common L3/L4 DDoS attacks (SYN floods, UDP reflection, etc.) on all AWS resources. Shield Advanced ($3,000/month per organization) adds: L7 DDoS protection, DDoS cost protection (credits for scaling charges during attacks), 24/7 Shield Response Team (SRT) access, advanced real-time metrics and reports, WAF fee waiver for protected resources, and health-based detection. Never claim Shield Standard provides L7 protection, cost protection, or SRT access.

## Guardrail 6: Body Inspection Limits — 8KB Default, Configurable to 64KB
By default, WAF inspects only the first 8 KB of the request body. This can be configured to 16 KB, 32 KB, or 64 KB for regional web ACLs (not CloudFront). Content beyond the inspection limit is NOT inspected — it passes through without rule evaluation. Increasing the body inspection limit increases WCU consumption. For CloudFront web ACLs, the limit is always 8 KB and cannot be changed.

## Guardrail 7: IP Set Size Limits — 10,000 Addresses Per Set
Each IP set can contain a maximum of 10,000 IP address CIDR ranges. IPv4 and IPv6 addresses must be in separate IP sets (each set is either IPV4 or IPV6). Use CIDR notation: /32 for single IPv4 addresses, /128 for single IPv6 addresses. IP sets are regional resources — a REGIONAL IP set cannot be used in a CLOUDFRONT web ACL and vice versa.

## Guardrail 8: Scope — REGIONAL vs CLOUDFRONT
REGIONAL scope web ACLs protect ALB, API Gateway REST API, AppSync GraphQL API, Cognito user pool, App Runner service, and Verified Access instance. CLOUDFRONT scope web ACLs protect CloudFront distributions only and MUST be created in us-east-1. You cannot attach a REGIONAL web ACL to CloudFront or a CLOUDFRONT web ACL to an ALB. All associated resources (IP sets, regex pattern sets, rule groups) must match the web ACL scope.

## Guardrail 9: Managed Rule Group Override Actions
When adding a managed rule group to a web ACL, use `OverrideAction` to control the group-level behavior: `None` (use the rule group's actions as-is) or `Count` (override all actions to Count for testing). For individual rules within the group, use `RuleActionOverrides` to override specific rules to Count while keeping others active. Always test new managed rule groups in Count mode before switching to Block.

## Guardrail 10: Label Namespace and Matching
Labels added by rules follow the format `awswaf:<entity>:<scope>:<label-name>`. AWS managed rule labels use `awswaf:managed:<vendor>:<rule-group>:<label>`. Custom labels use `awswaf:<account-id>:<web-acl>:<label>`. Labels exist only during the web ACL evaluation of a single request — they do not persist. Label match statements can only reference labels added by rules with lower priority numbers (evaluated earlier).

## Guardrail 11: Custom Response Body Limits
Custom response bodies are limited to 4 KB each. A web ACL can have a maximum of 50 custom response bodies. Supported content types are application/json, text/html, and text/plain. Custom responses can only be used with Block actions and custom request handling with Allow and Count actions. Custom response headers are limited to 5 per rule action.

## Guardrail 12: Shield Advanced Cost Protection Requires Route 53 Health Check
DDoS cost protection (automatic credits for scaling charges during attacks) requires that the protected resource has an associated Route 53 health check. Without the health check, Shield Advanced cannot verify that traffic spikes are DDoS attacks vs legitimate traffic surges, and cost protection claims will be denied. The health check must monitor the protected resource's availability. Always configure health checks before relying on cost protection.
