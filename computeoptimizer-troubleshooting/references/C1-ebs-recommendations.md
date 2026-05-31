---
title: "C1 — EBS Volume Recommendations"
description: "Diagnose EBS volume optimization recommendations"
status: active
severity: MEDIUM
triggers: ["EBS recommendation", "volume sizing", "IOPS optimization"]
owner: devops-agent
objective: "Resolve EBS recommendation issues"
context: "EBS recommendations analyze IOPS, throughput, and size utilization."
---

## Phase 1 — Triage

MUST:
- Get recommendations: `aws compute-optimizer get-ebs-volume-recommendations --volume-arns <arn>`
- Check volume utilization metrics
- Verify enrollment

## Phase 2 — Remediate

MUST:
- Review volume type change recommendations
- Test changes in non-production first

## Common Issues

- symptoms: "Recommendation suggests gp3 from gp2"
  diagnosis: "gp3 provides better price-performance for the workload."
  resolution: "Evaluate gp3 migration. Test performance. Migrate."

## Output Format

```yaml
root_cause: "ebs_recommendations — <specific_cause>"
evidence:
  - type: recommendation
    content: "<EBS recommendation>"
severity: MEDIUM
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Recommendations based on volume utilization metrics |
| Blast Radius | HIGH — Volume type changes can affect I/O performance and application stability |
| Rollback Complexity | MEDIUM — Volume type changes require modification and may have cooldown periods |
| Compliance Scope | LOW — Advisory recommendations only |

## Escalation Conditions

- Escalate to application owners before changing EBS volume types on production workloads
- Escalate to operations if volume modification causes I/O performance degradation
- Escalate to AWS Support if volume modification gets stuck

## Data Sensitivity

- **Volume ARNs and IOPS metrics** are standard infrastructure metadata
- No application data is exposed through EBS recommendations

## PROHIBITED Actions

- Do NOT change EBS volume types on production databases without a maintenance window
- Do NOT reduce provisioned IOPS below application requirements
- Do NOT modify volumes during peak I/O periods
- Do NOT ignore the 6-hour cooldown between EBS volume modifications

## Phase 3 — Rollback

- Modify the volume back to the previous type/IOPS (subject to 6-hour cooldown)
- Monitor VolumeReadOps, VolumeWriteOps, and latency metrics after rollback
- If immediate rollback is needed, create a new volume from snapshot with original configuration

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
