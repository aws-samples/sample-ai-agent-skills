---
title: "E2 — Origin Endpoint Issues"
description: "Diagnose MediaStore origin endpoint connectivity and configuration issues"
status: active
severity: HIGH
triggers: ["endpoint unreachable", "origin endpoint error", "cannot connect to mediastore", "endpoint configuration"]
owner: devops-agent
objective: "Resolve MediaStore origin endpoint connectivity and configuration issues"
context: "Each MediaStore container has a unique data plane endpoint. The endpoint is used for all object operations and as the CDN origin."
---

## Phase 1 — Triage

MUST:
- Get container endpoint: `aws mediastore describe-container --container-name <name> --query 'Container.{Endpoint:Endpoint,Status:Status}'`
- Test endpoint connectivity: `curl -I <container-endpoint>/`
- Verify container status is ACTIVE

SHOULD:
- Check if the endpoint resolves in DNS: `nslookup <endpoint-domain>`
- Verify the client can reach the endpoint (no firewall or proxy blocking)

MAY:
- Check for regional service issues in AWS Health Dashboard
- Test from different network locations to isolate connectivity issues

## Phase 2 — Remediate

MUST:
- Ensure the container is in ACTIVE status
- Use the correct endpoint URL (from describe-container, not constructed manually)
- Verify HTTPS is used for all endpoint connections

SHOULD:
- Configure retry logic in applications for transient endpoint failures
- Use the endpoint from the API response, not hardcoded values

MAY:
- Set up health checks for the endpoint in monitoring systems

## Safety Ratings

```
safety_ratings:
  - "mediastore describe-container: GREEN — read-only container query"
  - "curl -I: GREEN — read-only HTTP HEAD request"
  - "nslookup: GREEN — read-only DNS query"
  - "Endpoint health check configuration: GREEN — monitoring setup"
```

## Escalation Conditions

- Endpoint is unreachable from all locations (potential AWS service issue)
- Container is in ACTIVE status but endpoint returns 500 errors
- DNS resolution fails for the endpoint domain
- Endpoint latency exceeds SLA requirements for live streaming

## Data Sensitivity

- Container endpoint URL: LOW — required for data plane access
- DNS resolution results: LOW — IP addresses of AWS endpoints
- HTTP response headers: LOW — server headers and status codes
- Container status: LOW — operational state

## Prohibited Actions

- NEVER construct endpoint URLs manually — always use describe-container
- NEVER cache endpoint URLs indefinitely — they may change if container is recreated
- NEVER expose endpoint URLs with write permissions to untrusted clients
- NEVER use HTTP (non-SSL) to connect to MediaStore endpoints

## Phase 3 — Rollback

1. If endpoint was changed in application configuration, revert to previous value
2. If DNS or proxy settings were modified, restore original configuration
3. Verify endpoint connectivity is restored
4. Check that all dependent services (CloudFront, media players) can reach the endpoint
5. Document the connectivity issue and resolution

## Common Issues

- symptoms: "Endpoint returns 403 for all requests"
  diagnosis: "Container policy is too restrictive or IAM credentials are not being sent."
  resolution: "Check container policy. Ensure requests include valid AWS SigV4 authentication."

- symptoms: "Endpoint works from some locations but not others"
  diagnosis: "Network firewall or proxy is blocking HTTPS to the MediaStore endpoint domain."
  resolution: "Allow outbound HTTPS to *.data.mediastore.<region>.amazonaws.com."

## Output Format

```yaml
root_cause: "origin_endpoint — <specific_cause>"
evidence:
  - type: endpoint_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint configuration or connectivity"
  long_term: "Implement endpoint health monitoring"
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
