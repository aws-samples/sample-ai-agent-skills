---
title: "C2 — Endpoint DNS Issues"
description: "Diagnose DNS configuration problems for Verified Access endpoints"
status: active
severity: HIGH
triggers:
  - "DNS not resolving"
  - "endpoint DNS"
  - "cannot reach endpoint"
  - "domain not working"
owner: devops-agent
objective: "Identify and resolve DNS configuration issues for Verified Access endpoints"
context: "Each endpoint has a generated DNS name. Users access applications through a custom domain that CNAMEs to the endpoint DNS. Without proper DNS, users cannot reach the application. The endpoint domain prefix combined with the Verified Access domain forms the full endpoint DNS name."
---

## Phase 1 — Triage

MUST:
- Get endpoint DNS name: `aws ec2 describe-verified-access-endpoints --verified-access-endpoint-ids <endpoint-id> --query 'VerifiedAccessEndpoints[*].EndpointDomain'`
- Verify CNAME record exists: `dig CNAME <your-domain>`
- Test DNS resolution: `nslookup <your-domain>`
- Verify the CNAME target matches the endpoint DNS name

SHOULD:
- Check DNS propagation if recently changed
- Verify SSL certificate covers the custom domain
- Check TTL values on DNS records

MAY:
- Test endpoint DNS name directly (bypass custom domain)
- Check for DNS caching issues

## Phase 2 — Remediate

MUST:
- Create CNAME record: `<your-app-domain> CNAME <endpoint-dns-name>`
- Ensure SSL certificate (ACM) covers the custom domain
- Wait for DNS propagation

SHOULD:
- Use low TTL during initial setup for faster changes
- Verify HTTPS works end-to-end after DNS setup

MAY:
- Use Route 53 for DNS management
- Set up DNS health checks

## Common Issues

- symptoms: "Users cannot reach application through Verified Access"
  diagnosis: "CNAME record not created or pointing to wrong endpoint."
  resolution: "Create CNAME from app domain to endpoint DNS name."

- symptoms: "SSL certificate error in browser"
  diagnosis: "Certificate doesn't cover the custom domain."
  resolution: "Use ACM certificate that includes the custom domain name."

- symptoms: "DNS resolves but connection fails"
  diagnosis: "Endpoint not in active state or security group blocking."
  resolution: "Check endpoint status. Verify security groups."

## Output Format

```yaml
root_cause: "endpoint_dns — <specific_cause>"
evidence:
  - type: endpoint_domain
    content: "<endpoint DNS name>"
  - type: dns_resolution
    content: "<DNS lookup result>"
  - type: cname_record
    content: "<CNAME configuration>"
severity: HIGH
mitigation:
  immediate: "Create or fix CNAME record"
  long_term: "Document DNS requirements for each endpoint"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Trust provider configurations contain identity provider details
- Access logs reveal user access patterns and device information
- Cedar policies define authorization logic for applications
- Classification: CONFIDENTIAL - access control policies and user activity data

## Prohibited Actions

- NEVER modify trust provider configurations during active user sessions
- NEVER change Verified Access policies that could lock out production users
- NEVER delete endpoints serving active application traffic
- NEVER modify Cedar policies without testing in a non-production environment first

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
