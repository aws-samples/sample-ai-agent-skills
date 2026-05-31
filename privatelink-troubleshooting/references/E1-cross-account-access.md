---
title: "E1 — Cross-Account Access"
description: "Diagnose cross-account VPC endpoint access issues"
status: active
severity: HIGH
triggers:
  - "cross-account"
  - "different account"
  - "account not allowed"
  - "cross-account endpoint"
  - "shared endpoint service"
owner: devops-agent
objective: "Resolve cross-account access issues for VPC endpoint services"
context: "Cross-account PrivateLink requires the service provider to add the consumer account to the allowed principals list. The consumer creates an interface endpoint in their VPC pointing to the service. Connection acceptance may be required. Endpoint policies can further restrict cross-account access. IAM policies in both accounts must allow the operations."
---

## Phase 1 — Triage

MUST:
- Check endpoint service permissions (provider side): `aws ec2 describe-vpc-endpoint-service-permissions --service-id <service-id>`
- Check connection status: `aws ec2 describe-vpc-endpoint-connections --filters Name=vpc-endpoint-id,Values=<endpoint-id>`
- Verify the consumer account is in allowed principals
- Check endpoint state in consumer account: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id>`

SHOULD:
- Verify the service name used by the consumer matches the provider's service
- Check if connection acceptance is pending
- Verify IAM permissions in the consumer account for creating endpoints

MAY:
- Check endpoint policies for cross-account restrictions
- Verify network connectivity between consumer VPC and endpoint

## Phase 2 — Remediate

MUST:
- Add consumer account to allowed principals (provider side): `aws ec2 modify-vpc-endpoint-service-permissions --service-id <service-id> --add-allowed-principals arn:aws:iam::<consumer-account-id>:root`
- Accept the connection (provider side): `aws ec2 accept-vpc-endpoint-connections --service-id <service-id> --vpc-endpoint-ids <endpoint-id>`
- Consumer creates endpoint using the service name from the provider

SHOULD:
- Enable auto-accept for trusted accounts
- Configure endpoint policies for least-privilege cross-account access
- Document the cross-account setup process

MAY:
- Use AWS RAM for sharing endpoint services across accounts in an organization
- Implement automated acceptance workflows

## Common Issues

- symptoms: "Consumer cannot find the endpoint service"
  diagnosis: "Consumer account not in allowed principals list."
  resolution: "Provider must add consumer account to allowed principals."

- symptoms: "Endpoint created but connection pending"
  diagnosis: "Manual acceptance required by the provider."
  resolution: "Provider must accept the connection."

- symptoms: "Cross-account endpoint policy blocking access"
  diagnosis: "Endpoint policy restricts which principals can use the endpoint."
  resolution: "Update endpoint policy to allow the cross-account principal."

## Output Format

```yaml
root_cause: "cross_account — <specific_cause>"
evidence:
  - type: allowed_principals
    content: "<endpoint service allowed principals>"
  - type: connection_status
    content: "<connection acceptance status>"
  - type: endpoint_policy
    content: "<endpoint policy document>"
severity: HIGH
mitigation:
  immediate: "Add consumer to allowed principals and accept connection"
  long_term: "Automate cross-account endpoint provisioning"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — cross-account access can expose services to unauthorized accounts
- Service Disruption Risk: MEDIUM — access issues prevent legitimate cross-account connectivity
- Compliance Impact: HIGH — cross-account access must comply with organizational policies

## Escalation Conditions
- Unauthorized accounts gain access to endpoint services
- Cross-account access fails for authorized accounts
- Acceptance settings modified without authorization
- Cross-account traffic patterns indicate unauthorized usage
- Organizational policies block required cross-account access

## Data Sensitivity
- Account IDs and access configurations — CONFIDENTIAL
- Cross-account trust relationships — CONFIDENTIAL
- Endpoint service permissions — CONFIDENTIAL
- Traffic patterns — INTERNAL
- Never expose cross-account access configurations externally

## Prohibited Actions
- Do NOT grant cross-account access without proper authorization
- Do NOT auto-accept connections from unverified accounts
- Do NOT bypass cross-account access review processes
- Do NOT share endpoint service details with unauthorized accounts
- Do NOT ignore unauthorized cross-account access attempts
- Do NOT modify cross-account permissions without security review

## Phase 3 — Rollback
- Revoke unauthorized cross-account access immediately
- Restore previous acceptance configurations
- Revert permission changes
- Verify authorized access after changes
- Document cross-account access changes
- Update cross-account access procedures

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
