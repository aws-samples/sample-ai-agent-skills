---
title: "G1 — VPC Configuration Issues"
description: "Diagnose VPC networking problems for Verified Access endpoints"
status: active
severity: HIGH
triggers:
  - "VPC configuration"
  - "subnet error"
  - "endpoint VPC"
  - "network configuration"
owner: devops-agent
objective: "Identify and resolve VPC configuration issues for Verified Access"
context: "Verified Access endpoints are deployed in VPC subnets. The endpoint must be able to reach the backend application. Subnets, route tables, and security groups must be properly configured. Endpoints create ENIs in the specified subnets."
---

## Phase 1 — Triage

MUST:
- Check endpoint VPC configuration: `aws ec2 describe-verified-access-endpoints --verified-access-endpoint-ids <endpoint-id>`
- Verify subnets exist and have available IPs: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Check route tables for subnets: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify backend is reachable from endpoint subnets

SHOULD:
- Check for NACL rules blocking traffic
- Verify endpoint ENIs are created: `aws ec2 describe-network-interfaces --filters Name=description,Values="*verified-access*"`
- Check VPC DNS settings

MAY:
- Check VPC Flow Logs for traffic analysis
- Verify cross-AZ connectivity if endpoint and backend are in different AZs

## Phase 2 — Remediate

MUST:
- Ensure subnets have available IP addresses
- Verify route tables allow traffic to backend
- Fix NACLs if blocking traffic

SHOULD:
- Place endpoints in multiple AZs for high availability
- Ensure subnets have routes to the internet (for OIDC authentication redirects)

MAY:
- Use VPC endpoints for AWS service access
- Implement VPC Flow Logs for monitoring

## Common Issues

- symptoms: "Endpoint cannot reach backend"
  diagnosis: "Route table missing route to backend subnet or NACL blocking."
  resolution: "Add route to backend. Check NACLs allow traffic."

- symptoms: "Endpoint creation fails with subnet error"
  diagnosis: "Subnet has no available IPs or is in wrong VPC."
  resolution: "Use subnet with available IPs in the correct VPC."

- symptoms: "Authentication redirect fails"
  diagnosis: "Endpoint subnet has no internet access for OIDC redirect."
  resolution: "Ensure subnet has route to internet (NAT gateway or internet gateway)."

## Output Format

```yaml
root_cause: "vpc_config — <specific_cause>"
evidence:
  - type: subnet_config
    content: "<subnet details and available IPs>"
  - type: route_tables
    content: "<route table entries>"
  - type: nacls
    content: "<NACL rules>"
severity: HIGH
mitigation:
  immediate: "Fix VPC networking configuration"
  long_term: "Document VPC requirements for Verified Access"
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

- Trust provider configurations contain identity provider details
- Access logs reveal user access patterns and device information
- Cedar policies define authorization logic for applications
- Classification: CONFIDENTIAL - access control policies and user activity data

## Prohibited Actions

- NEVER modify trust provider configurations during active user sessions
- NEVER change Verified Access policies that could lock out production users
- NEVER delete endpoints serving active application traffic
- NEVER modify Cedar policies without testing in a non-production environment first

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
