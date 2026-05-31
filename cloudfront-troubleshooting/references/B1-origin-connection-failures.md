---
title: "B1 — Origin Connection Failures"
description: "Diagnose CloudFront failures to connect to the origin server"
status: active
severity: CRITICAL
triggers:
  - "502 Bad Gateway"
  - "504 Gateway Timeout"
  - "origin connection"
  - "origin unreachable"
  - "OriginConnectError"
owner: devops-agent
objective: "Restore connectivity between CloudFront and the origin"
context: "CloudFront returns 502 when it cannot establish a TCP connection or receives an invalid response from the origin. It returns 504 when the origin does not respond within the configured timeout. Common causes include origin down, firewall blocking CloudFront IPs, SSL handshake failures, and incorrect origin domain/port."
---

## Phase 1 — Triage

MUST:
- Check distribution origin configuration: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins'`
- Check 5xx error rate: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 5xxErrorRate --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check origin latency: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name OriginLatency --dimensions Name=DistributionId,Value=<dist-id> Name=Region,Value=Global --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify origin is reachable directly: `curl -sI https://<origin-domain>:<port>/<path>`
- Check origin connection timeout and read timeout settings in the distribution config

SHOULD:
- Verify origin security group allows CloudFront IP ranges (if EC2/ALB origin)
- Check origin health if behind a load balancer
- Verify DNS resolution for the origin domain: `nslookup <origin-domain>`

MAY:
- Check CloudFront IP ranges: `curl -s https://ip-ranges.amazonaws.com/ip-ranges.json | jq '.prefixes[] | select(.service=="CLOUDFRONT")'`
- Check origin server logs for connection attempts from CloudFront

## Phase 2 — Remediate

MUST:
- Fix origin availability if the origin is down
- Correct origin domain name, port, and protocol in distribution config
- Adjust connection timeout (1-10 seconds) and read timeout (1-60 seconds) if origin is slow

SHOULD:
- Configure origin failover with an origin group for high availability
- Whitelist CloudFront IP ranges in origin firewall/security group
- Use origin custom headers to verify requests come from CloudFront

MAY:
- Enable Origin Shield to reduce origin load and improve availability
- Implement health checks on the origin

## Common Issues

- symptoms: "502 Bad Gateway intermittently"
  diagnosis: "Origin server is overloaded or has intermittent connectivity issues."
  resolution: "Scale origin, increase timeouts, configure origin failover."

- symptoms: "504 Gateway Timeout on large responses"
  diagnosis: "Origin read timeout (default 30s) is too short for slow responses."
  resolution: "Increase origin read timeout up to 60 seconds."

- symptoms: "502 after changing origin to HTTPS"
  diagnosis: "SSL handshake failure — origin certificate not trusted or protocol mismatch."
  resolution: "Check origin SSL certificate. Set origin protocol policy to match origin capabilities."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
```

## Escalation Conditions

- Distribution serves a production website or application
- Fix requires modifying origin configuration or cache behaviors
- Resolution involves certificate changes or HTTPS configuration
- Issue affects multiple distributions or is account-level
- Lambda@Edge or CloudFront Functions changes are required on production

## Data Sensitivity

MEDIUM - Signed URL private keys and key pairs control content access. Origin configurations may expose internal infrastructure (S3 bucket names, ALB endpoints). Access logs contain client IPs, request URIs, and query strings. Field-level encryption configurations protect sensitive form data.

## Prohibited Actions

- NEVER suggest deleting a CloudFront distribution that is serving live traffic
- NEVER suggest disabling HTTPS or downgrading the security policy on a production distribution
- NEVER recommend removing all cache behaviors - this breaks content routing
- NEVER suggest invalidating '/*' repeatedly as a fix - address the root caching issue instead
- NEVER recommend removing origin access control/identity from S3 origins without alternative access controls

## Phase 3 - Rollback

1. If distribution configuration was changed, update with previous settings: `aws cloudfront update-distribution --id <id> --distribution-config <previous> --if-match <etag>`
2. If SSL/TLS settings were changed, restore original certificate and security policy configuration
3. If origin configuration was changed, restore original origin settings including timeouts and protocols
4. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "origin_connection — <specific_cause>"
evidence:
  - type: origin_config
    content: "<origin domain, port, protocol>"
  - type: error_rate
    content: "<5xx error rate metrics>"
  - type: origin_latency
    content: "<origin latency metrics>"
severity: CRITICAL
mitigation:
  immediate: "Restore origin connectivity"
  long_term: "Configure origin failover and Origin Shield"
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
  - "NEVER suggest disabling HTTPS requirements"
  - "NEVER suggest removing WAF association to fix access"
  - "NEVER suggest wildcard CORS origins in production"
