---
title: "C3 — Endpoint Connectivity Issues"
description: "Diagnose connectivity problems between users and Verified Access endpoints"
status: active
severity: HIGH
triggers:
  - "endpoint not reachable"
  - "connection timeout"
  - "502 error"
  - "503 error"
owner: devops-agent
objective: "Identify and resolve endpoint connectivity issues"
context: "Connectivity issues can occur between users and the endpoint (DNS, certificate, network) or between the endpoint and the backend application (security groups, backend health, VPC routing). Both paths must work for end-to-end connectivity."
---

## Phase 1 — Triage

MUST:
- Check endpoint status: `aws ec2 describe-verified-access-endpoints --verified-access-endpoint-ids <endpoint-id> --query 'VerifiedAccessEndpoints[*].Status'`
- Verify DNS resolves correctly
- Check security groups on endpoint: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify backend application is healthy

SHOULD:
- Check if the issue is authentication (redirect to IdP) or connectivity (timeout/error)
- Verify endpoint security group allows inbound HTTPS (443)
- Check backend security group allows traffic from endpoint security group

MAY:
- Check VPC Flow Logs for blocked traffic
- Test backend directly (bypassing Verified Access) to isolate the issue

## Phase 2 — Remediate

MUST:
- Ensure endpoint is in active status
- Fix security groups: inbound 443 on endpoint, outbound to backend
- Verify backend is healthy and responding

SHOULD:
- Check that backend returns valid HTTP responses
- Verify SSL certificates are valid on both endpoint and backend

MAY:
- Implement health monitoring for the endpoint
- Set up alerts for endpoint status changes

## Common Issues

- symptoms: "502 Bad Gateway"
  diagnosis: "Backend not responding or returning invalid response."
  resolution: "Check backend health. Verify backend port and protocol match endpoint config."

- symptoms: "Connection timeout"
  diagnosis: "Security group blocking or endpoint not active."
  resolution: "Check endpoint status. Verify security groups allow traffic."

- symptoms: "Redirect loop"
  diagnosis: "Authentication configuration issue with IdP."
  resolution: "Check trust provider OIDC configuration. Verify redirect URIs."

## Output Format

```yaml
root_cause: "endpoint_connectivity — <specific_cause>"
evidence:
  - type: endpoint_status
    content: "<endpoint state>"
  - type: security_groups
    content: "<security group rules>"
  - type: backend_health
    content: "<backend application status>"
severity: HIGH
mitigation:
  immediate: "Fix connectivity path (DNS → endpoint → backend)"
  long_term: "Implement end-to-end monitoring"
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
