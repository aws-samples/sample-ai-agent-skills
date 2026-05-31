---
title: "B3 — Performance"
description: "Diagnose and act on Trusted Advisor performance recommendations"
status: active
severity: MEDIUM
triggers:
  - "performance check"
  - "performance recommendation"
  - "high utilization"
  - "service limits"
owner: devops-agent
objective: "Investigate and act on performance recommendations"
context: "Performance checks identify high-utilization resources, service limit warnings, CloudFront optimization opportunities, and EC2 instance type recommendations. Service limit checks compare current usage against quotas."
---

## Phase 1 — Triage

MUST:
- Get performance check summaries: `aws support describe-trusted-advisor-check-summaries --check-ids <check-ids> --region us-east-1`
- Get detailed performance findings: `aws support describe-trusted-advisor-check-result --check-id <check-id> --language en --region us-east-1`
- Verify service limit warnings against actual quotas: `aws service-quotas list-service-quotas --service-code <service>`
- Check resource utilization metrics

SHOULD:
- Cross-reference service limits with Service Quotas: `aws service-quotas get-service-quota --service-code <code> --quota-code <code>`
- Check if flagged resources are actually experiencing performance issues
- Review CloudWatch metrics for utilization trends

MAY:
- Check if performance issues correlate with specific time periods
- Review auto-scaling configurations for flagged resources

## Phase 2 — Remediate

MUST:
- Request quota increases for service limits approaching thresholds
- Right-size resources showing consistently high utilization
- Address performance bottlenecks identified by checks

SHOULD:
- Set up CloudWatch alarms for utilization thresholds
- Implement auto-scaling for variable workloads
- Request proactive quota increases before hitting limits

MAY:
- Implement performance monitoring dashboards
- Set up automated scaling policies

## Common Issues

- symptoms: "Service limit at 80% or above"
  diagnosis: "Resource usage approaching service quota limit."
  resolution: "Request quota increase: `aws service-quotas request-service-quota-increase --service-code <code> --quota-code <code> --desired-value <value>`"

- symptoms: "High utilization EC2 instances"
  diagnosis: "Instances consistently running at high CPU/memory utilization."
  resolution: "Upgrade instance type or implement auto-scaling."

- symptoms: "Service limit warning but quota was increased"
  diagnosis: "Trusted Advisor check hasn't refreshed since quota increase."
  resolution: "Refresh the check. TA may lag behind Service Quotas updates."

## Output Format

```yaml
root_cause: "performance — <specific_cause>"
evidence:
  - type: performance_findings
    content: "<flagged resources and utilization>"
  - type: service_limits
    content: "<current usage vs quota>"
severity: MEDIUM
mitigation:
  immediate: "Request quota increases and right-size resources"
  long_term: "Implement proactive monitoring and auto-scaling"
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
