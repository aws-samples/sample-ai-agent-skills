---
title: "Z1 — General Wavelength Troubleshooting (Catch-All)"
description: "Fallback SOP for Wavelength issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Wavelength issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check Wavelength Zone status: `aws ec2 describe-availability-zones --filters "Name=zone-type,Values=wavelength-zone" --all-availability-zones`
- List Wavelength subnets: `aws ec2 describe-subnets --filters "Name=availability-zone,Values=<wavelength-zone>"`
- Check carrier gateway: `aws ec2 describe-carrier-gateways`
- List instances in Wavelength: `aws ec2 describe-instances --filters "Name=availability-zone,Values=<wavelength-zone>"`
- Check carrier IPs: `aws ec2 describe-addresses --filters "Name=domain,Values=vpc" --query 'Addresses[?CarrierIp!=null]'`

SHOULD:
- Check route tables for Wavelength subnets
- Verify security groups and NACLs
- Check CloudTrail for recent changes
- Review CloudWatch metrics for anomalies

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Zone opt-in or carrier gateway → Zone (A1-A2)
- EC2 or ECS/EKS issues → Compute (B1-B2)
- Carrier IP or subnet issues → Networking (C1-C2)
- Latency or throughput → Performance (D1-D2)
- Region connectivity or S3 → Integration (E1-E2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all Wavelength configurations systematically
- Review CloudTrail for recent configuration changes
- Test connectivity end-to-end (mobile → Wavelength → Region)
- Check AWS Health Dashboard for Wavelength service events

SHOULD:
- Compare with known-good Wavelength deployment
- Check carrier network status
- Review application logs on Wavelength instances

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<zone|compute|networking|performance|integration|unknown>"
investigation_path: "zone status → carrier gateway → instances → <domain_classification>"
evidence:
  - type: zone_config
    content: "<Wavelength Zone configuration>"
  - type: networking
    content: "<carrier gateway and routing>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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

- Edge deployment configurations reveal application architecture
- Carrier IP assignments expose public-facing endpoints
- Latency data may reveal user geographic distribution
- Classification: INTERNAL - edge infrastructure topology and carrier network details

## Prohibited Actions

- NEVER terminate EC2 instances in Wavelength zones serving live edge traffic
- NEVER modify carrier gateway configurations during peak usage
- NEVER change VPC subnet associations without confirming application connectivity
- NEVER remove carrier IP addresses assigned to production workloads

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
