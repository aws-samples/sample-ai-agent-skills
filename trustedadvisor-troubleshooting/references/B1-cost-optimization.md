---
title: "B1 — Cost Optimization"
description: "Diagnose and act on Trusted Advisor cost optimization recommendations"
status: active
severity: MEDIUM
triggers:
  - "cost optimization check"
  - "idle resources"
  - "underutilized instances"
  - "cost savings recommendation"
owner: devops-agent
objective: "Investigate and validate cost optimization recommendations"
context: "Cost optimization checks identify idle resources, underutilized instances, unassociated Elastic IPs, idle load balancers, and RI optimization opportunities. Requires Business/Enterprise Support for full checks."
---

## Phase 1 — Triage

MUST:
- Get cost optimization check summaries: `aws support describe-trusted-advisor-check-summaries --check-ids Qch7DwouX1 Z4AUBRNSmz hjLMh88uM8 DAvU99Dc4C --region us-east-1`
- Get detailed results for flagged checks: `aws support describe-trusted-advisor-check-result --check-id <check-id> --language en --region us-east-1`
- Verify flagged resources are actually idle/underutilized
- Cross-reference with CloudWatch metrics for utilization data

SHOULD:
- Check EC2 instance utilization: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=<id> --start-time <iso> --end-time <iso> --period 86400 --statistics Average`
- Verify EBS volumes are unattached: `aws ec2 describe-volumes --volume-ids <id> --query 'Volumes[].{State:State,Attachments:Attachments}'`
- Check for unassociated Elastic IPs: `aws ec2 describe-addresses --query 'Addresses[?AssociationId==null]'`

MAY:
- Calculate potential savings from recommendations
- Check if resources are part of disaster recovery or standby configurations

## Phase 2 — Remediate

MUST:
- Validate each recommendation before acting (some resources may be intentionally idle)
- Document business justification for keeping flagged resources
- Terminate or downsize confirmed idle/underutilized resources

SHOULD:
- Set up automated idle resource detection
- Implement resource lifecycle policies
- Create cost optimization review schedule

MAY:
- Implement automated remediation for specific resource types
- Set up cost optimization dashboards

## Common Issues

- symptoms: "Low utilization EC2 instances flagged"
  diagnosis: "Instances running but CPU utilization consistently below 10%."
  resolution: "Verify workload. Right-size or terminate if truly idle."

- symptoms: "Unassociated Elastic IPs"
  diagnosis: "EIPs allocated but not associated with running instances. Incurs hourly charge."
  resolution: "Associate with instance or release: `aws ec2 release-address --allocation-id <id>`"

- symptoms: "Idle load balancers"
  diagnosis: "Load balancer with no healthy targets or minimal traffic."
  resolution: "Verify if still needed. Delete if no longer required."

## Output Format

```yaml
root_cause: "cost_optimization — <specific_cause>"
evidence:
  - type: check_results
    content: "<flagged resources and estimated savings>"
  - type: utilization_data
    content: "<CloudWatch utilization metrics>"
severity: MEDIUM
mitigation:
  immediate: "Validate and act on confirmed idle resources"
  long_term: "Implement automated idle resource detection and lifecycle policies"
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
