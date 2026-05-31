---
title: "A1 — EC2 Instance Recommendations"
description: "Diagnose missing or incorrect EC2 recommendations"
status: active
severity: MEDIUM
triggers: ["no EC2 recommendations", "wrong instance type", "EC2 sizing"]
owner: devops-agent
objective: "Resolve EC2 recommendation issues"
context: "EC2 recommendations analyze CPU, memory, network, and disk metrics."
---

## Phase 1 — Triage

MUST:
- Check enrollment: `aws compute-optimizer get-enrollment-status`
- Get recommendations: `aws compute-optimizer get-ec2-instance-recommendations --instance-arns <arn>`
- Verify instance has 14+ days of data
- Check finding classification

SHOULD:
- Verify CloudWatch agent for memory metrics
- Check for enhanced metrics enrollment

## Phase 2 — Remediate

MUST:
- Enroll if not opted in
- Wait for data if new instance
- Install CloudWatch agent for memory

## Common Issues

- symptoms: "No recommendations for running instance"
  diagnosis: "Instance too new or account not enrolled."
  resolution: "Verify enrollment. Wait 14+ days. Check instance state."

## Output Format

```yaml
root_cause: "ec2_recommendations — <specific_cause>"
evidence:
  - type: recommendation
    content: "<EC2 recommendation details>"
severity: MEDIUM
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Recommendations are based on aggregated CloudWatch metrics, not application data |
| Blast Radius | HIGH — Acting on incorrect recommendations can cause performance degradation or outages |
| Rollback Complexity | MEDIUM — Instance type changes require stop/start and may cause downtime |
| Compliance Scope | LOW — Recommendations are advisory; no direct compliance impact |

## Escalation Conditions

- Escalate to application owners before applying any instance type changes to production
- Escalate to management if recommendations suggest significant cost changes (>20% increase or decrease)
- Escalate to AWS Support if recommendations are consistently inaccurate despite sufficient data

## Data Sensitivity

- **Recommendation data** contains instance ARNs and utilization metrics — low sensitivity
- **CloudWatch metrics** used for recommendations are standard infrastructure metrics
- **Account enrollment status** is operational metadata

## PROHIBITED Actions

- Do NOT apply EC2 instance type changes to production without testing in staging first
- Do NOT blindly follow recommendations without understanding application requirements
- Do NOT change instance types during peak traffic periods
- Do NOT ignore memory metrics — install CloudWatch agent for complete recommendations

## Phase 3 — Rollback

- Stop the instance and change back to the previous instance type
- Restart the instance and verify application performance
- Monitor CloudWatch metrics for 24 hours after rollback to confirm stability
- Document the failed recommendation for future reference

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
