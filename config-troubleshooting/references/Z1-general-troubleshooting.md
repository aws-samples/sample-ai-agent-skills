---
title: "Z1 — General AWS Config Troubleshooting (Catch-All)"
description: "Fallback SOP for AWS Config issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown AWS Config issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check recorder status: `aws configservice describe-configuration-recorder-status`
- Check delivery channel status: `aws configservice describe-delivery-channel-status`
- List Config rules: `aws configservice describe-config-rules`
- Check compliance summary: `aws configservice describe-compliance-by-config-rule`
- List conformance packs: `aws configservice describe-conformance-packs`
- Check aggregators: `aws configservice describe-configuration-aggregators`

SHOULD:
- Check CloudWatch Config metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Config --metric-name ConfigRulesEvaluationsFailed --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check CloudTrail for Config events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=config.amazonaws.com --max-results 20`
- List remediation configurations: `aws configservice describe-remediation-configurations --config-rule-names <rule-name>`
- Check organization rules: `aws configservice describe-organization-config-rules`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Rule evaluation failures → Rules (A1-A3)
- Conformance pack issues → Conformance (B1-B2)
- Aggregator problems → Aggregator (C1-C2)
- Remediation failures → Remediation (D1-D2)
- Recorder or delivery issues → Recorder (E1-E3)
- Query errors → Advanced Query (F1-F2)
- Organization/multi-account → Multi-Account (G1-G2)
- Integration problems → Integration (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check the Config service role permissions: `aws iam get-role --role-name <config-role>`
- Verify the S3 bucket for delivery: `aws s3api head-bucket --bucket <config-bucket>`
- Check for recent configuration changes in CloudTrail
- Verify resource type recording coverage: `aws configservice get-discovered-resource-counts`

SHOULD:
- Check AWS Health Dashboard for Config service events
- Verify no SCPs are blocking Config operations
- Check service quotas for Config rules and configuration items
- Review recent changes to IAM roles used by Config

MAY:
- Check Organizations configuration if using multi-account features
- Verify KMS key status for encrypted delivery channels
- Review Lambda function health for custom rules

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Route to specific SOP | GREEN | Classification only; no state change |
| Investigate and collect evidence | GREEN | Read-only analysis |

## Escalation Conditions

- Recorder configuration changes
- Remediation action deployment
- Unclassified issues after investigation

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-configuration-recorder-status` | LOW | Recorder status |
| `describe-delivery-channel-status` | LOW | Delivery status |
| `describe-config-rules` | LOW | Rule configuration |
| `describe-compliance-by-config-rule` | LOW | Compliance summary |
| CloudTrail events | MEDIUM | API call history |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Route to specific SOP | Follow that SOP's rollback procedures |
| No direct remediation in catch-all | N/A — remediation happens in specific SOPs |

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<rules|conformance|aggregator|remediation|recorder|query|multi_account|integration|unknown>"
investigation_path: "recorder → delivery → rules → compliance → <domain_classification>"
evidence:
  - type: recorder
    content: "<recorder status summary>"
  - type: delivery
    content: "<delivery channel status>"
  - type: rules
    content: "<rule compliance summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```

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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
