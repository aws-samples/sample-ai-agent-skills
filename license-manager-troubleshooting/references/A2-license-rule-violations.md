---
title: "A2 — License Rule Violations"
description: "Diagnose license rule violations and compliance issues"
status: active
severity: CRITICAL
triggers: ["license violation", "license exceeded", "compliance violation", "license limit reached"]
owner: devops-agent
objective: "Resolve license rule violations and restore compliance"
context: "License rule violations occur when resource consumption exceeds configured limits. Hard limits block launches; soft limits generate alerts."
---

## Phase 1 — Triage

MUST:
- Check license usage: `aws license-manager list-usage-for-license-configuration --license-configuration-arn <arn>`
- Get configuration details: `aws license-manager get-license-configuration --license-configuration-arn <arn>`
- List associated resources: `aws license-manager list-associations-for-license-configuration --license-configuration-arn <arn>`

SHOULD:
- Check CloudWatch for license consumption metrics
- Identify which resources are consuming the most licenses
- Verify counting type is correct for the software

MAY:
- Review historical consumption trends
- Check if terminated instances are still counted

## Phase 2 — Remediate

MUST:
- Identify over-consuming resources and right-size or terminate
- If licenses are legitimately needed, procure additional licenses and update count: `aws license-manager update-license-configuration --license-configuration-arn <arn> --license-count <new-count>`
- Verify counting type matches vendor agreement

SHOULD:
- Set up CloudWatch alarms at 80% consumption threshold
- Review and remove stale resource associations

MAY:
- Implement automated scaling policies that respect license limits
- Set up License Manager reports for regular compliance reviews

## Safety Ratings

```
safety_ratings:
  - "license-manager list-usage-for-license-configuration: GREEN — read-only usage check"
  - "license-manager get-license-configuration: GREEN — read-only config check"
  - "license-manager update-license-configuration: YELLOW — modifies license limits"
  - "ec2 terminate-instances: RED — terminates running instances"
```

## Escalation Conditions

- License violation affects production workloads
- Hard limit blocks critical resource launches
- Vendor audit is imminent and violations exist
- License procurement requires budget approval

## Data Sensitivity

- License usage data: MEDIUM — reveals software consumption patterns
- Resource associations: MEDIUM — reveals infrastructure details
- Compliance status: HIGH — audit-relevant information
- Vendor license counts: HIGH — contractual information

## Prohibited Actions

- NEVER increase license counts beyond purchased quantities to hide violations
- NEVER disable hard limits during an active audit
- NEVER terminate production instances to reduce license count without approval
- NEVER delete violation history or compliance records

## Phase 3 — Rollback

1. If license count was increased incorrectly, revert: `aws license-manager update-license-configuration --license-configuration-arn <arn> --license-count <original-count>`
2. If resources were terminated to reduce consumption, verify service impact
3. If counting type was changed, revert to the correct type
4. Document all changes made during violation remediation
5. Notify compliance team of the violation and resolution

## Common Issues

- symptoms: "EC2 launch fails with LicenseLimitExceeded"
  diagnosis: "Hard limit enforcement is blocking launches because license count is exceeded."
  resolution: "Procure additional licenses and update count, or terminate unused instances to free licenses."

## Output Format

```yaml
root_cause: "license_violation — <specific_cause>"
evidence:
  - type: violation_details
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Resolve license violation"
  long_term: "Implement proactive license monitoring"
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
