---
title: "A3 — Sampling Rules Issues"
description: "Diagnose X-Ray sampling rule configuration problems"
status: active
severity: MEDIUM
triggers:
  - "sampling not working"
  - "too many traces"
  - "too few traces"
  - "sampling rule not applying"
owner: devops-agent
objective: "Identify and fix sampling rule configuration issues"
context: "Sampling rules control which requests are traced and at what rate. Rules are evaluated in priority order (lowest first). The default rule catches unmatched requests. Misconfigured rules cause over-sampling (cost) or under-sampling (missing data)."
---

## Phase 1 — Triage

MUST:
- List all sampling rules: `aws xray get-sampling-rules`
- Check sampling statistics: `aws xray get-sampling-statistic-summaries`
- Identify the rule priority order and which rules match the affected service
- Verify the reservoir size and fixed rate for matching rules

SHOULD:
- Check if a higher-priority rule (lower number) is matching before the intended rule
- Verify the rule's service name, URL path, and HTTP method filters
- Compare configured rates with actual sampling statistics

MAY:
- Test with a temporary 100% sampling rule to confirm traces appear
- Check if the SDK is correctly reporting sampling decisions

## Phase 2 — Remediate

MUST:
- Fix rule priorities to ensure correct evaluation order
- Adjust reservoir size and fixed rate for desired sampling behavior
- Update rule filters (service name, URL path, HTTP method) to match correctly

SHOULD:
- Create specific rules for high-priority services:
  `aws xray create-sampling-rule --sampling-rule '{"RuleName":"my-rule","Priority":100,"FixedRate":0.1,"ReservoirSize":5,"ServiceName":"my-service","ServiceType":"*","Host":"*","ResourceARN":"*","HTTPMethod":"*","URLPath":"*","Version":1}'`
- Remove or disable conflicting rules

MAY:
- Set up different sampling rates for different environments
- Monitor sampling statistics regularly

## Common Issues

- symptoms: "All requests being traced (high cost)"
  diagnosis: "Default rule or custom rule with 100% rate matching all traffic."
  resolution: "Reduce the fixed rate. Use reservoir for guaranteed minimum samples."

- symptoms: "Specific service not being sampled"
  diagnosis: "Higher-priority rule with 0% rate matching before the intended rule."
  resolution: "Check rule priorities. Adjust or remove the blocking rule."

- symptoms: "Sampling rule created but not taking effect"
  diagnosis: "Rule filters don't match the service name or URL path."
  resolution: "Verify ServiceName, URLPath, and HTTPMethod match the actual request attributes."

## Output Format

```yaml
root_cause: "sampling_rule_issue — <specific_cause>"
evidence:
  - type: sampling_rules
    content: "<all rules with priorities>"
  - type: sampling_statistics
    content: "<actual sampling rates>"
  - type: rule_match
    content: "<which rule matches the affected traffic>"
severity: MEDIUM
mitigation:
  immediate: "Fix sampling rule configuration"
  long_term: "Document sampling strategy and manage rules via IaC"
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

- Trace data reveals application request flow and dependencies
- Segment metadata may contain request/response details
- Service maps expose complete application architecture
- Classification: CONFIDENTIAL - application architecture and request flow data

## Prohibited Actions

- NEVER modify sampling rules that could cause production trace data loss
- NEVER delete X-Ray groups used by production monitoring dashboards
- NEVER change encryption settings without coordinating with security team
- NEVER disable tracing on production services without approval

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
