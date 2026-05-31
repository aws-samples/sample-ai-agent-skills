---
title: "G1 — Endpoint Policies (Security)"
description: "Diagnose and audit VPC endpoint policies for security compliance"
status: active
severity: MEDIUM
triggers:
  - "endpoint policy audit"
  - "data exfiltration"
  - "restrict S3 access"
  - "endpoint security"
  - "policy compliance"
owner: devops-agent
objective: "Audit and secure endpoint policies to prevent data exfiltration and enforce compliance"
context: "Endpoint policies are a critical security control for preventing data exfiltration through VPC endpoints. The default policy allows all principals and all actions. Restrictive policies can limit which S3 buckets, DynamoDB tables, or other resources can be accessed through the endpoint. Endpoint policies are commonly used in regulated environments."
---

## Phase 1 — Triage

MUST:
- List all endpoints and their policies: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> --query 'VpcEndpoints[*].{Id:VpcEndpointId,Service:ServiceName,Policy:PolicyDocument}'`
- Identify endpoints with default (full access) policies
- Check if endpoint policies restrict access to authorized resources only
- Verify endpoint policies align with security requirements

SHOULD:
- Check for S3 endpoint policies that restrict bucket access
- Verify endpoint policies prevent access to unauthorized accounts' resources
- Review endpoint policies for overly permissive conditions
- Check AWS Config rules for endpoint policy compliance

MAY:
- Audit endpoint policies across all VPCs in the account
- Compare endpoint policies with organizational security standards

## Phase 2 — Remediate

MUST:
- Replace default policies with restrictive policies on sensitive endpoints
- Restrict S3 endpoint to specific buckets: limit to organization-owned buckets
- Include condition keys (aws:PrincipalOrgID, aws:ResourceOrgID) for organization-level restrictions

SHOULD:
- Implement endpoint policies via IaC for consistency
- Set up AWS Config rules to detect non-compliant endpoint policies
- Document required endpoint policies for each service

MAY:
- Implement automated policy remediation via Lambda
- Create endpoint policy templates for common use cases

## Common Issues

- symptoms: "Data exfiltration risk through S3 endpoint"
  diagnosis: "S3 endpoint policy allows access to any S3 bucket, including external accounts."
  resolution: "Restrict endpoint policy to organization-owned buckets using aws:ResourceOrgID condition."

- symptoms: "Endpoint policy too restrictive — blocking legitimate access"
  diagnosis: "Policy missing required actions or resources."
  resolution: "Add the required actions and resource ARNs to the endpoint policy."

- symptoms: "No endpoint policies configured — all using defaults"
  diagnosis: "Default policies allow unrestricted access through endpoints."
  resolution: "Implement restrictive policies on all endpoints, especially S3 and DynamoDB."

## Output Format

```yaml
root_cause: "endpoint_policy_security — <specific_cause>"
evidence:
  - type: policy_audit
    content: "<endpoints with default vs restrictive policies>"
  - type: compliance_gaps
    content: "<identified security gaps>"
  - type: recommendations
    content: "<recommended policy changes>"
severity: MEDIUM
mitigation:
  immediate: "Apply restrictive policies to high-risk endpoints"
  long_term: "Implement policy management, compliance monitoring, and drift detection"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — endpoint policies control data access through endpoints
- Service Disruption Risk: MEDIUM — overly restrictive policies block legitimate access
- Compliance Impact: HIGH — endpoint policies enforce data access compliance

## Escalation Conditions
- Endpoint policy changes block critical service access
- Overly permissive policies detected
- Policy evaluation errors cause inconsistent access
- Compliance audit flags endpoint policy violations
- Policy changes affect multiple endpoints simultaneously

## Data Sensitivity
- Endpoint policy documents — CONFIDENTIAL
- Access control configurations — CONFIDENTIAL
- Policy evaluation logs — INTERNAL
- Service access patterns — INTERNAL
- Never expose endpoint policies that reveal security architecture

## Prohibited Actions
- Do NOT apply overly permissive endpoint policies
- Do NOT remove endpoint policies without understanding impact
- Do NOT bypass endpoint policy review processes
- Do NOT create policies that allow unauthorized data access
- Do NOT ignore policy evaluation errors
- Do NOT modify policies during active incidents without approval

## Phase 3 — Rollback
- Restore previous endpoint policies
- Revert policy changes that blocked access
- Verify service access after policy changes
- Restore compliance-required policy controls
- Document policy changes and their impact
- Update endpoint policy management procedures

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
