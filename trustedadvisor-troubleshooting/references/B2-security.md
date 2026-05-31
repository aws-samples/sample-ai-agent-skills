---
title: "B2 — Security"
description: "Diagnose and act on Trusted Advisor security recommendations"
status: active
severity: HIGH
triggers:
  - "security check"
  - "security finding"
  - "open security group"
  - "security recommendation"
owner: devops-agent
objective: "Investigate and remediate security findings from Trusted Advisor"
context: "Security checks identify open security groups, public S3 buckets, missing MFA, exposed access keys, and other security risks. Core security checks (SG, S3, IAM, MFA, EBS/RDS snapshots) are available on all support plans."
---

## Phase 1 — Triage

MUST:
- Get security check summaries: `aws support describe-trusted-advisor-check-summaries --check-ids HCP4007jGY Pfx0RwqBli zXCkfM1nI3 7DAFEmoDos --region us-east-1`
- Get detailed security findings: `aws support describe-trusted-advisor-check-result --check-id <check-id> --language en --region us-east-1`
- Prioritize findings by severity (unrestricted access > missing MFA > exposed keys)
- Verify flagged resources are actually at risk

SHOULD:
- Check security groups for unrestricted access: `aws ec2 describe-security-groups --filters Name=ip-permission.cidr,Values=0.0.0.0/0 --query 'SecurityGroups[].{GroupId:GroupId,GroupName:GroupName}'`
- Verify S3 bucket public access: `aws s3api get-public-access-block --bucket <bucket>`
- Check IAM credential report: `aws iam generate-credential-report && aws iam get-credential-report`

MAY:
- Cross-reference with AWS Config compliance
- Check Security Hub for additional context

## Phase 2 — Remediate

MUST:
- Restrict overly permissive security groups immediately
- Enable MFA on root account and IAM users
- Rotate or deactivate exposed access keys

SHOULD:
- Enable S3 Block Public Access at account level
- Implement security group change monitoring
- Set up automated remediation for common findings

MAY:
- Integrate with Security Hub for centralized security management
- Implement preventive controls via SCPs

## Common Issues

- symptoms: "Security groups with unrestricted access (0.0.0.0/0)"
  diagnosis: "Security group allows inbound traffic from any IP on sensitive ports."
  resolution: "Restrict to specific CIDR ranges. Remove 0.0.0.0/0 rules for non-public ports."

- symptoms: "MFA not enabled on root account"
  diagnosis: "Root account lacks multi-factor authentication."
  resolution: "Enable MFA on root account immediately. Use virtual or hardware MFA device."

- symptoms: "Public EBS or RDS snapshots"
  diagnosis: "Snapshots shared publicly, exposing data."
  resolution: "Make snapshots private: `aws ec2 modify-snapshot-attribute --snapshot-id <id> --attribute createVolumePermission --operation-type remove --group-names all`"

## Output Format

```yaml
root_cause: "security — <specific_cause>"
evidence:
  - type: security_findings
    content: "<flagged resources and risk level>"
  - type: resource_config
    content: "<current security configuration>"
severity: HIGH
mitigation:
  immediate: "Remediate critical security findings"
  long_term: "Implement preventive controls and automated remediation"
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
