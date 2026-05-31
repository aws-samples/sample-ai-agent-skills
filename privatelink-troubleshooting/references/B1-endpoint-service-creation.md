---
title: "B1 — Endpoint Service Creation"
description: "Diagnose VPC endpoint service creation issues"
status: active
severity: HIGH
triggers:
  - "endpoint service creation"
  - "create endpoint service"
  - "service configuration"
  - "cannot create service"
  - "endpoint service failed"
owner: devops-agent
objective: "Resolve endpoint service creation and configuration issues"
context: "Endpoint services expose applications via PrivateLink. They require an NLB (or GWLB). The service provider creates the endpoint service and controls which accounts can connect. Endpoint services can have private DNS names (requires domain verification). Multiple NLBs can be associated for multi-AZ."
---

## Phase 1 — Triage

MUST:
- Check existing endpoint service configurations: `aws ec2 describe-vpc-endpoint-service-configurations`
- Verify NLB exists and is active: `aws elbv2 describe-load-balancers --load-balancer-arns <nlb-arn> --query 'LoadBalancers[*].{Name:LoadBalancerName,Type:Type,State:State.Code}'`
- Verify NLB type is "network" (not "application"): `aws elbv2 describe-load-balancers --load-balancer-arns <nlb-arn> --query 'LoadBalancers[*].Type'`
- Check IAM permissions for creating endpoint services

SHOULD:
- Verify NLB has healthy targets: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Check NLB listeners are configured: `aws elbv2 describe-listeners --load-balancer-arn <nlb-arn>`
- Verify NLB is in the same region as the endpoint service

MAY:
- Check if private DNS name domain verification is needed
- Review NLB cross-zone load balancing settings

## Phase 2 — Remediate

MUST:
- Create endpoint service with NLB: `aws ec2 create-vpc-endpoint-service-configuration --network-load-balancer-arns <nlb-arn> --acceptance-required`
- Verify the NLB is type "network" before associating
- Configure acceptance settings (manual or auto-accept)

SHOULD:
- Add allowed principals: `aws ec2 modify-vpc-endpoint-service-permissions --service-id <service-id> --add-allowed-principals <principal-arns>`
- Enable auto-accept if appropriate: `aws ec2 modify-vpc-endpoint-service-configuration --service-id <service-id> --no-acceptance-required`
- Configure private DNS name if needed

MAY:
- Associate multiple NLBs for multi-AZ redundancy
- Set up notifications for new connection requests

## Common Issues

- symptoms: "Cannot create endpoint service — ALB specified"
  diagnosis: "Endpoint services require NLB, not ALB."
  resolution: "Create an NLB. If ALB features needed, place ALB behind NLB."

- symptoms: "Endpoint service created but no one can connect"
  diagnosis: "No accounts added to allowed principals list."
  resolution: "Add consumer account ARNs to allowed principals."

- symptoms: "Private DNS name verification pending"
  diagnosis: "Domain ownership verification required for private DNS names."
  resolution: "Add the TXT record to your DNS zone as specified in the verification details."

## Output Format

```yaml
root_cause: "endpoint_service_creation — <specific_cause>"
evidence:
  - type: nlb_config
    content: "<NLB type, state, and listeners>"
  - type: service_config
    content: "<endpoint service configuration>"
  - type: permissions
    content: "<allowed principals list>"
severity: HIGH
mitigation:
  immediate: "Fix NLB or service configuration"
  long_term: "Automate endpoint service setup with IaC"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — endpoint services expose internal services
- Service Disruption Risk: MEDIUM — service creation issues affect service availability
- Compliance Impact: HIGH — endpoint services control cross-account access

## Escalation Conditions
- Endpoint service creation fails with NLB configuration errors
- Service becomes unavailable after creation
- Acceptance settings misconfigured allowing unauthorized access
- Service DNS configuration fails
- Multiple endpoint services fail simultaneously

## Data Sensitivity
- NLB configurations — INTERNAL
- Service endpoint details — INTERNAL
- Acceptance configurations — CONFIDENTIAL
- Cross-account access settings — CONFIDENTIAL
- Never expose endpoint service details to unauthorized parties

## Prohibited Actions
- Do NOT create endpoint services without proper acceptance controls
- Do NOT auto-accept connections from unknown accounts
- Do NOT expose endpoint services without NLB health checks
- Do NOT bypass endpoint service security review
- Do NOT share endpoint service names publicly
- Do NOT create endpoint services without monitoring

## Phase 3 — Rollback
- Delete incorrectly configured endpoint services
- Restore NLB configurations
- Revert acceptance settings
- Verify service availability after changes
- Document service creation issues
- Update endpoint service procedures

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
