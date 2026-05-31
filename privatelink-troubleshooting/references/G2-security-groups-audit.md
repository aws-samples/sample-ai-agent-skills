---
title: "G2 — Security Groups Audit"
description: "Audit and secure security groups on VPC interface endpoint ENIs"
status: active
severity: MEDIUM
triggers:
  - "security group audit"
  - "endpoint ENI security"
  - "overly permissive security group"
  - "security group compliance"
  - "endpoint access control"
owner: devops-agent
objective: "Audit and secure security groups on interface endpoint ENIs"
context: "Interface endpoint ENIs have security groups that control network-level access. Overly permissive security groups (0.0.0.0/0 inbound) expose the endpoint to all VPC traffic. Security groups should restrict inbound access to only authorized clients. Security groups are stateful — return traffic is automatically allowed."
---

## Phase 1 — Triage

MUST:
- List all endpoint ENIs and their security groups: `aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=<endpoint-id> --query 'NetworkInterfaces[*].{Id:NetworkInterfaceId,Groups:Groups}'`
- Check security group rules for overly permissive access: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Identify security groups allowing 0.0.0.0/0 inbound
- Verify security groups are consistent across all endpoint ENIs

SHOULD:
- Check if the default VPC security group is used (may be shared with other resources)
- Verify security group rules match the principle of least privilege
- Check for unused or stale security group rules

MAY:
- Audit security groups across all endpoints in the VPC
- Compare security group configurations with organizational standards

## Phase 2 — Remediate

MUST:
- Replace overly permissive rules with specific CIDR or security group references
- Use dedicated security groups for endpoint ENIs (not the default VPC security group)
- Restrict inbound to only the service port from authorized client CIDRs/security groups

SHOULD:
- Use security group references (sg-to-sg) instead of CIDR blocks where possible
- Apply consistent security groups across all ENIs of the same endpoint
- Document security group requirements for each endpoint

MAY:
- Implement security group management via IaC
- Set up AWS Config rules for security group compliance
- Enable VPC Flow Logs to monitor traffic patterns

## Common Issues

- symptoms: "Endpoint security group allows all inbound traffic"
  diagnosis: "Security group has 0.0.0.0/0 inbound rule."
  resolution: "Restrict inbound to specific client CIDRs or security groups on the service port only."

- symptoms: "Default VPC security group used for endpoint"
  diagnosis: "Default security group may be shared with other resources and have unintended rules."
  resolution: "Create a dedicated security group for the endpoint with specific rules."

- symptoms: "Inconsistent security groups across endpoint ENIs"
  diagnosis: "Different ENIs of the same endpoint have different security groups."
  resolution: "Apply the same security group to all ENIs of the endpoint."

## Output Format

```yaml
root_cause: "security_group_audit — <specific_cause>"
evidence:
  - type: security_groups
    content: "<endpoint ENI security group rules>"
  - type: compliance_gaps
    content: "<overly permissive or inconsistent rules>"
  - type: recommendations
    content: "<recommended security group changes>"
severity: MEDIUM
mitigation:
  immediate: "Fix overly permissive security group rules"
  long_term: "Implement security group management via IaC and compliance monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — audit is a review activity
- Service Disruption Risk: LOW — auditing does not affect endpoint operations
- Compliance Impact: HIGH — security group auditing is a compliance requirement

## Escalation Conditions
- Audit reveals unauthorized security group rules
- Audit findings indicate compliance violations
- Security group drift detected from approved baseline
- Audit unable to complete due to access issues
- Critical findings require immediate remediation

## Data Sensitivity
- Audit results — CONFIDENTIAL
- Security group baselines — INTERNAL
- Compliance findings — CONFIDENTIAL
- Remediation plans — INTERNAL
- Never expose audit findings externally without authorization

## Prohibited Actions
- Do NOT skip security group audits on production endpoints
- Do NOT ignore audit findings
- Do NOT delay remediation of critical findings
- Do NOT share audit results with unauthorized parties
- Do NOT bypass audit procedures
- Do NOT modify security groups to pass audits without proper fixes

## Phase 3 — Rollback
- Restore security groups to compliant baseline
- Revert unauthorized rule changes found in audit
- Re-run audit after remediation
- Verify compliance after changes
- Document audit findings and remediation steps
- Update security group baselines

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
