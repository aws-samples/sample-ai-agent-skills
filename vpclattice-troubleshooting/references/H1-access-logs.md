---
title: "H1 — Access Log Issues"
description: "Diagnose VPC Lattice access log configuration and delivery problems"
status: active
severity: MEDIUM
triggers:
  - "no access logs"
  - "missing logs"
  - "access log subscription"
  - "log delivery failed"
owner: devops-agent
objective: "Resolve access log configuration and delivery issues"
context: "VPC Lattice access logs capture request/response details including auth decisions, matched rules, and target health. Logs must be explicitly enabled via access log subscriptions on the service network. Destinations: CloudWatch Logs, S3, or Kinesis Data Firehose."
---

## Phase 1 — Triage

MUST:
- Check access log subscriptions: `aws vpc-lattice list-access-log-subscriptions --resource-identifier <service-network-id>`
- Get subscription details: `aws vpc-lattice get-access-log-subscription --access-log-subscription-identifier <id>`
- Verify destination exists (CloudWatch log group, S3 bucket, or Firehose stream)
- Check IAM permissions for VPC Lattice to write to destination

SHOULD:
- Verify log group/bucket policy allows vpc-lattice.amazonaws.com
- Check if subscription is in ACTIVE state
- Look for delivery errors in CloudTrail

## Phase 2 — Remediate

MUST:
- Create access log subscription if none exists: `aws vpc-lattice create-access-log-subscription --resource-identifier <sn-id> --destination-arn <dest-arn>`
- Ensure destination resource policy allows VPC Lattice service principal
- Verify subscription reaches ACTIVE state

SHOULD:
- Use CloudWatch Logs for real-time analysis
- Use S3 for long-term storage and Athena queries
- Set appropriate log retention policies

## Output Format

```yaml
root_cause: "access-logs — <specific_cause>"
evidence:
  - type: subscription_status
    content: "<ACTIVE or missing>"
  - type: destination
    content: "<log destination ARN>"
severity: MEDIUM
mitigation:
  immediate: "Create or fix access log subscription"
  long_term: "Automate log subscription creation with IaC"
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

- Service network configurations reveal microservice architecture
- Auth policies contain inter-service access control rules
- Access logs reveal service-to-service communication patterns
- Classification: INTERNAL - service mesh topology and access patterns

## Prohibited Actions

- NEVER delete service networks with active service associations
- NEVER modify auth policies that could disrupt inter-service communication
- NEVER remove target groups serving production traffic
- NEVER change listener rules without testing routing behavior first

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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
