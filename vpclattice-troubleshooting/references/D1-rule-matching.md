---
title: "D1 — Listener Rule Matching Issues"
description: "Diagnose listener rule matching and priority problems"
status: active
severity: MEDIUM
triggers:
  - "rule not matching"
  - "wrong target group"
  - "rule priority"
  - "path matching"
owner: devops-agent
objective: "Resolve listener rule matching issues"
context: "Listener rules match requests by path and/or header conditions. Rules evaluate in priority order (lowest number first). The default rule (priority 0) catches unmatched requests. Misconfigured rules cause traffic to reach wrong targets."
---

## Phase 1 — Triage

MUST:
- List rules: `aws vpc-lattice list-rules --service-identifier <svc-id> --listener-identifier <listener-id>`
- Get rule details: `aws vpc-lattice get-rule --service-identifier <svc-id> --listener-identifier <listener-id> --rule-identifier <rule-id>`
- Check rule priority order and match conditions
- Verify target groups referenced by rules exist and have healthy targets

SHOULD:
- Check access logs for which rule matched: look at `matchedRulePriority` field
- Verify path patterns use correct syntax (exact, prefix, or contains)

## Phase 2 — Remediate

MUST:
- Ensure rule priorities are ordered correctly (most specific first)
- Verify path match conditions use correct match type
- Confirm target groups in rule actions are healthy

SHOULD:
- Use access logs to verify rule matching behavior
- Test with curl to confirm expected routing

## Output Format

```yaml
root_cause: "rule-matching — <specific_cause>"
evidence:
  - type: rule_list
    content: "<rules with priorities>"
  - type: match_conditions
    content: "<path/header conditions>"
severity: MEDIUM
mitigation:
  immediate: "Fix rule priority or match conditions"
  long_term: "Document routing rules and test after changes"
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
