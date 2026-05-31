---
title: "D1 — EventBridge"
description: "Diagnose Trusted Advisor EventBridge integration issues"
status: active
severity: MEDIUM
triggers:
  - "EventBridge not receiving TA events"
  - "TA notification not working"
  - "check status event missing"
  - "EventBridge integration"
owner: devops-agent
objective: "Resolve Trusted Advisor EventBridge integration issues"
context: "Trusted Advisor publishes check status change events to EventBridge. Events contain check-level status changes, not resource details. EventBridge rules must match the correct event pattern. Events are published in us-east-1."
---

## Phase 1 — Triage

MUST:
- Check EventBridge rules for TA events: `aws events list-rules --name-prefix TrustedAdvisor --region us-east-1`
- Verify rule event pattern matches Trusted Advisor source: `aws events describe-rule --name <rule-name> --region us-east-1`
- Check rule targets: `aws events list-targets-by-rule --rule <rule-name> --region us-east-1`
- Verify rule is enabled

SHOULD:
- Check if events are being published (trigger a check refresh and monitor)
- Verify target permissions (SNS topic policy, Lambda invoke permission)
- Check CloudWatch metrics for EventBridge rule invocations

MAY:
- Review EventBridge dead-letter queue for failed deliveries
- Check CloudTrail for EventBridge API events

## Phase 2 — Remediate

MUST:
- Create EventBridge rule with correct pattern:
  ```json
  {"source": ["aws.trustedadvisor"], "detail-type": ["Trusted Advisor Check Item Refresh Notification"]}
  ```
- Ensure rule is in us-east-1 (TA events publish here)
- Configure target with appropriate permissions

SHOULD:
- Add target dead-letter queue for failed deliveries
- Set up Lambda target to fetch resource details via API
- Test with a manual check refresh

MAY:
- Forward events to other regions via EventBridge cross-region rules
- Implement custom event processing pipeline

## Common Issues

- symptoms: "No events received from Trusted Advisor"
  diagnosis: "Rule in wrong region or event pattern incorrect."
  resolution: "Create rule in us-east-1 with source aws.trustedadvisor."

- symptoms: "Events received but no resource details"
  diagnosis: "TA events contain check status, not resource details."
  resolution: "Use Lambda target to call describe-trusted-advisor-check-result for details."

- symptoms: "Target not invoked despite events"
  diagnosis: "Target permissions missing (Lambda invoke, SNS publish)."
  resolution: "Add resource-based policy allowing events.amazonaws.com to invoke target."

## Output Format

```yaml
root_cause: "eventbridge — <specific_cause>"
evidence:
  - type: rule_config
    content: "<EventBridge rule and event pattern>"
  - type: target_config
    content: "<target configuration and permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix rule pattern, region, or target permissions"
  long_term: "Implement comprehensive event processing pipeline"
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

- Check results reveal security vulnerabilities and misconfigurations
- Cost data exposes spending patterns and resource utilization
- Organizational view aggregates findings across all accounts
- Classification: CONFIDENTIAL - security posture and cost intelligence data

## Prohibited Actions

- NEVER dismiss security-critical Trusted Advisor findings without documented justification
- NEVER disable Trusted Advisor checks required by compliance frameworks
- NEVER ignore cost optimization recommendations exceeding defined thresholds without review
- NEVER modify organizational view settings without administrator approval

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
