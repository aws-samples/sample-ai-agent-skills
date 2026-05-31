---
title: "Z1 — General VPC IPAM Troubleshooting"
description: "Catch-all runbook for VPC IPAM issues"
status: active
severity: MEDIUM
triggers: ["IPAM error", "IPAM not working", "IP address management problem"]
owner: devops-agent
objective: "Systematic investigation of unclassified VPC IPAM issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List IPAMs: `aws ec2 describe-ipams --query 'Ipams[*].{IpamId:IpamId,State:State,OperatingRegions:OperatingRegions}'`
- Check pools: `aws ec2 describe-ipam-pools --query 'IpamPools[*].{PoolId:IpamPoolId,State:State,Locale:Locale}'`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20`

SHOULD:
- Check resource discovery status
- Verify IPAM scopes and pool hierarchy

MAY:
- Check AWS Health Dashboard for EC2/IPAM service issues
- Review CloudWatch metrics for IPAM

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for recurring issues

## Safety Ratings

```
safety_ratings:
  - "describe-ipams: GREEN — read-only IPAM enumeration"
  - "describe-ipam-pools: GREEN — read-only pool listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "describe-ipam-scopes: GREEN — read-only scope listing"
```

## Escalation Conditions

- IPAM service appears non-functional in the region
- Multiple unrelated IPAM operations fail simultaneously
- CloudTrail shows unauthorized API calls to IPAM
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws ec2 describe-ipams`: LOW — returns IPAM metadata
- `aws ec2 describe-ipam-pools`: LOW — returns pool metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- IPAM resource data: MEDIUM — reveals IP address management structure

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without first identifying the root cause
- NEVER delete IPAMs, pools, or scopes as a troubleshooting step
- NEVER dismiss the issue without documenting the investigation path and findings
- NEVER modify pool allocations or CIDRs during investigation

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore IPAM, pool, and scope configurations to pre-investigation state
3. Verify all IPAM operations are functioning normally
4. Document all investigation steps and findings for the network team
5. Verify resource discovery and compliance monitoring are operational

## Output Format

```yaml
root_cause: "general_ipam — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and alerting"
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
