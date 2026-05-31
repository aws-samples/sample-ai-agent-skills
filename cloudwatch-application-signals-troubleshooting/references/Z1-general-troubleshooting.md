---
title: "Z1 — General Application Signals Troubleshooting"
description: "Catch-all runbook for CloudWatch Application Signals issues"
status: active
severity: MEDIUM
triggers: ["Application Signals error", "Application Signals not working", "application monitoring problem"]
owner: devops-agent
objective: "Systematic investigation of unclassified Application Signals issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List services: `aws application-signals list-services --start-time $(date -d '-1 hour' -u +%s) --end-time $(date -u +%s)`
- Check SLOs: `aws application-signals list-service-level-objectives`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=application-signals.amazonaws.com --max-results 20`

SHOULD:
- Check metrics in AWS/ApplicationSignals namespace
- Verify CloudWatch agent and ADOT agent status

MAY:
- Check AWS Health Dashboard for Application Signals service issues
- Review CloudWatch Logs for agent errors

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for recurring issues

## Safety Ratings

```
safety_ratings:
  - "list-services: GREEN — read-only service enumeration"
  - "list-service-level-objectives: GREEN — read-only SLO listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "cloudwatch list-metrics: GREEN — read-only metric listing"
```

## Escalation Conditions

- Application Signals service appears non-functional in the region
- Multiple unrelated Application Signals features fail simultaneously
- CloudTrail shows unauthorized API calls to Application Signals
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws application-signals list-services`: LOW — returns service metadata
- `aws application-signals list-service-level-objectives`: LOW — returns SLO metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- Agent logs: MEDIUM — may contain application and infrastructure details

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without first identifying the root cause
- NEVER delete SLOs, services, or agent configurations as a troubleshooting step
- NEVER dismiss the issue without documenting the investigation path and findings
- NEVER disable instrumentation on production services during investigation

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore agent configurations to pre-investigation state
3. Verify all Application Signals features are functioning normally
4. Document all investigation steps and findings
5. Verify SLO evaluations are not affected by investigation activities

## Output Format

```yaml
root_cause: "general_application_signals — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement comprehensive monitoring"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
