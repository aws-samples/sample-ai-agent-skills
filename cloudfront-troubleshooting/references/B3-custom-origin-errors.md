---
title: "B3 — Custom Origin Errors"
description: "Diagnose errors from custom origins (ALB, EC2, API Gateway, on-premises)"
status: active
severity: HIGH
triggers:
  - "custom origin"
  - "ALB origin"
  - "EC2 origin"
  - "API Gateway origin"
  - "origin 502"
  - "origin 503"
owner: devops-agent
objective: "Resolve errors between CloudFront and custom origins"
context: "Custom origins include ALB, EC2, API Gateway, and any HTTP/HTTPS endpoint. CloudFront connects to custom origins using configurable protocol, port, timeouts, and custom headers. Common issues include protocol mismatch, Host header forwarding, keep-alive timeouts, and origin overload."
---

## Phase 1 — Triage

MUST:
- Check origin configuration: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins.Items[*].{Id:Id,Domain:DomainName,CustomOrigin:CustomOriginConfig,CustomHeaders:CustomHeaders}'`
- Check origin protocol policy (HTTP Only, HTTPS Only, Match Viewer)
- Check origin HTTP/HTTPS ports
- Verify origin is reachable: `curl -sI -H "Host: <expected-host>" https://<origin-domain>:<port>/<path>`
- Check 5xx error rate and origin latency in CloudWatch

SHOULD:
- Verify the Host header forwarded to origin matches what the origin expects
- Check origin keep-alive timeout (must be greater than CloudFront's 5-second keep-alive)
- Verify origin security group allows CloudFront IP ranges
- Check origin health check status if behind ALB

MAY:
- Test with origin custom headers to verify CloudFront is the source
- Check origin server access logs for CloudFront requests

## Phase 2 — Remediate

MUST:
- Match origin protocol policy to what the origin supports
- Ensure origin keep-alive timeout > 5 seconds to avoid connection reuse errors
- Forward the correct Host header via origin request policy or cache policy

SHOULD:
- Set appropriate connection timeout (1-10s) and read timeout (1-60s)
- Configure origin custom headers for origin verification
- Use HTTPS between CloudFront and origin for security

MAY:
- Implement origin failover with an origin group
- Use Origin Shield to reduce origin connections

## Common Issues

- symptoms: "502 Bad Gateway with ALB origin"
  diagnosis: "ALB idle timeout (default 60s) is less than CloudFront read timeout, or ALB security group blocks CloudFront."
  resolution: "Set ALB idle timeout > CloudFront read timeout. Allow CloudFront IPs in security group."

- symptoms: "Origin returns wrong content or 404"
  diagnosis: "Host header not forwarded — origin receives CloudFront domain instead of expected hostname."
  resolution: "Forward Host header to origin via origin request policy."

- symptoms: "Intermittent 502 errors"
  diagnosis: "Origin keep-alive timeout < CloudFront's 5-second keep-alive, causing connection reuse on closed connections."
  resolution: "Set origin keep-alive timeout to at least 6 seconds."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
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
2. If cache policy or TTL was changed, restore original cache behavior settings and allow caches to repopulate
3. If SSL/TLS settings were changed, restore original certificate and security policy configuration
4. If origin configuration was changed, restore original origin settings including timeouts and protocols
5. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "custom_origin — <specific_cause>"
evidence:
  - type: origin_config
    content: "<protocol, port, timeout settings>"
  - type: error_rate
    content: "<5xx error rate>"
  - type: origin_response
    content: "<direct origin test result>"
severity: HIGH
mitigation:
  immediate: "Fix origin configuration or connectivity"
  long_term: "Implement origin failover and proper timeout alignment"
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
