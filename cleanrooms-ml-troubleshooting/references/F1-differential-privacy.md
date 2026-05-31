---
title: "F1 — Differential Privacy"
description: "Diagnose differential privacy configuration, epsilon settings, and noise-related issues"
status: active
severity: HIGH
triggers: ["differential privacy", "epsilon", "noise too high", "privacy parameter", "utility tradeoff"]
owner: devops-agent
objective: "Resolve differential privacy configuration issues and optimize privacy-utility tradeoff"
context: "Differential privacy adds mathematical noise to outputs to protect individual records. Epsilon controls the privacy-utility tradeoff. Configuration affects audience quality and privacy guarantees."
---

## Phase 1 — Triage

MUST:
- Check configured audience model privacy settings: `aws cleanroomsml get-configured-audience-model --configured-audience-model-arn <arn>`
- Review audience generation job output quality metrics
- Verify epsilon and other privacy parameters in the collaboration configuration

SHOULD:
- Compare output quality across different audience generation jobs
- Check if privacy budget consumption aligns with expected epsilon values

MAY:
- Review collaboration agreement for agreed-upon privacy parameters
- Check CloudTrail for privacy parameter change events

## Phase 2 — Remediate

MUST:
- Verify epsilon value is within the range agreed upon by collaboration members
- Ensure privacy parameters are consistent across all configured audience models
- Document the privacy-utility tradeoff for the current configuration

SHOULD:
- Test different epsilon values with small audience generation jobs to find optimal tradeoff
- Coordinate with collaboration members before changing privacy parameters

MAY:
- Consult differential privacy documentation for parameter guidance
- Implement monitoring for output quality metrics across jobs

## Common Issues

- symptoms: "Audience output has too much noise — results are not useful"
  diagnosis: "Epsilon value is too low, adding excessive noise for strong privacy."
  resolution: "Increase epsilon (with collaboration member agreement) to reduce noise. Note: this weakens privacy guarantees."

- symptoms: "Privacy guarantees questioned by collaboration member"
  diagnosis: "Epsilon value may be too high, providing insufficient privacy protection."
  resolution: "Review epsilon against regulatory requirements. Decrease epsilon if privacy guarantees are insufficient."

## Safety Ratings

```
safety_ratings:
  - "get-configured-audience-model: GREEN — read-only privacy parameter retrieval"
  - "get-audience-generation-job: GREEN — read-only job details"
  - "update-configured-audience-model: YELLOW — changes privacy parameters"
  - "start-audience-generation-job: RED — consumes privacy budget with current parameters"
```

## Escalation Conditions

- Collaboration members disagree on epsilon values
- Regulatory requirements mandate specific privacy parameters
- Output quality is unacceptable at any privacy-compliant epsilon level
- Privacy budget exhaustion prevents testing different configurations

## Data Sensitivity

- `aws cleanroomsml get-configured-audience-model`: MEDIUM — reveals privacy parameters
- Privacy parameter values: MEDIUM — reveals collaboration privacy posture
- Audience output quality metrics: LOW — operational metrics
- Epsilon configuration history: MEDIUM — reveals privacy parameter evolution

## Prohibited Actions

- NEVER recommend specific epsilon values without understanding the use case and regulatory context
- NEVER change epsilon without agreement from ALL collaboration members
- NEVER suggest disabling differential privacy to improve output quality
- NEVER claim specific epsilon values guarantee regulatory compliance

## Phase 3 — Rollback

1. If epsilon was changed incorrectly, update configured audience model with previous value
2. Audience generation jobs already run with incorrect epsilon cannot be undone — budget was consumed
3. Document the parameter change and its impact on output quality
4. Notify all collaboration members of the privacy parameter change
5. Review any audience outputs generated with incorrect parameters

## Output Format

```yaml
root_cause: "differential_privacy — <specific_cause>"
evidence:
  - type: privacy_config
    content: "<privacy parameter details and output quality metrics>"
severity: HIGH
mitigation:
  immediate: "Adjust epsilon with member agreement"
  long_term: "Establish privacy parameter governance process"
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
