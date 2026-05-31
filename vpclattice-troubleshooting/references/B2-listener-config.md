---
title: "B2 — Listener Configuration Issues"
description: "Diagnose VPC Lattice listener and protocol configuration problems"
status: active
severity: HIGH
triggers:
  - "listener not working"
  - "protocol mismatch"
  - "listener creation failed"
  - "port configuration"
owner: devops-agent
objective: "Resolve listener configuration issues"
context: "Listeners define the protocol and port for incoming requests. VPC Lattice supports HTTP, HTTPS, and TLS_PASSTHROUGH listeners. Each listener has a default action and optional rules."
---

## Phase 1 — Triage

MUST:
- List listeners: `aws vpc-lattice list-listeners --service-identifier <id>`
- Get listener details: `aws vpc-lattice get-listener --service-identifier <id> --listener-identifier <id>`
- Check listener protocol and port configuration
- Verify default action target group exists and is healthy

SHOULD:
- Check if HTTPS listener has valid ACM certificate
- Verify TLS_PASSTHROUGH is used only when end-to-end encryption is needed

## Phase 2 — Remediate

MUST:
- Ensure listener protocol matches client expectations (HTTP/HTTPS)
- Verify default action points to valid target group
- For HTTPS: ensure ACM certificate covers the service domain

SHOULD:
- Use HTTPS for production services
- Configure appropriate default action (forward or fixed-response)

## Output Format

```yaml
root_cause: "listener-config — <specific_cause>"
evidence:
  - type: listener_details
    content: "<protocol, port, default action>"
  - type: certificate_status
    content: "<ACM cert status if HTTPS>"
severity: HIGH
mitigation:
  immediate: "Fix listener protocol or certificate"
  long_term: "Standardize listener configurations"
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

- Service network configurations reveal microservice architecture
- Auth policies contain inter-service access control rules
- Access logs reveal service-to-service communication patterns
- Classification: INTERNAL - service mesh topology and access patterns

## Prohibited Actions

- NEVER delete service networks with active service associations
- NEVER modify auth policies that could disrupt inter-service communication
- NEVER remove target groups serving production traffic
- NEVER change listener rules without testing routing behavior first

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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
