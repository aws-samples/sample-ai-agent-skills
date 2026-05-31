---
title: "A1 — Cloud Map DNS Namespace Issues"
description: "Diagnose DNS namespace creation and configuration problems"
status: active
severity: HIGH
triggers:
  - "DNS namespace"
  - "private namespace"
  - "public namespace"
  - "namespace creation failed"
  - "hosted zone"
owner: devops-agent
objective: "Identify and resolve DNS namespace issues in Cloud Map"
context: "DNS namespaces create Route 53 hosted zones (public or private). Private namespaces require VPC association. Namespace creation failures stem from naming conflicts, VPC issues, or IAM permissions."
---

## Phase 1 — Triage

MUST:
- List namespaces: `aws servicediscovery list-namespaces`
- Check namespace details: `aws servicediscovery get-namespace --id <namespace-id>`
- Verify namespace type (DNS_PUBLIC, DNS_PRIVATE, or HTTP)
- Check associated Route 53 hosted zone: `aws route53 list-hosted-zones --query 'HostedZones[?Name==\`<namespace-name>.\`]'`
- Check CloudTrail for creation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreatePrivateDnsNamespace --max-results 10`

SHOULD:
- For private namespaces, verify VPC association: `aws route53 get-hosted-zone --id <zone-id> --query 'VPCs'`
- Check for naming conflicts with existing namespaces
- Verify operation status: `aws servicediscovery get-operation --operation-id <operation-id>`

MAY:
- Check Route 53 hosted zone record count
- Verify DNS resolution from within the VPC

## Phase 2 — Remediate

MUST:
- For private namespaces, specify a valid VPC ID at creation
- Use unique namespace names to avoid conflicts
- Grant servicediscovery and route53 permissions to the calling identity

SHOULD:
- Associate additional VPCs if needed: `aws route53 associate-vpc-with-hosted-zone --hosted-zone-id <zone-id> --vpc VPCRegion=<region>,VPCId=<vpc-id>`
- Enable DNS hostnames and DNS support on the VPC
- Tag namespaces for resource management

MAY:
- Use CloudFormation or Terraform for namespace provisioning
- Set up monitoring for namespace health

## Common Issues

- symptoms: "Namespace creation fails with VPC error"
  diagnosis: "VPC doesn't exist, DNS support disabled, or VPC in wrong region."
  resolution: "Verify VPC exists with DNS support enabled in the same region."

- symptoms: "Namespace created but DNS not resolving"
  diagnosis: "Private hosted zone not associated with the client VPC."
  resolution: "Associate the hosted zone with the VPC where clients need DNS resolution."

- symptoms: "Namespace name conflict"
  diagnosis: "A namespace with the same name already exists."
  resolution: "Use a different name or delete the existing namespace."

## Output Format

```yaml
root_cause: "dns_namespace_issue — <specific_cause>"
evidence:
  - type: namespace_config
    content: "<namespace type and details>"
  - type: hosted_zone
    content: "<Route 53 hosted zone details>"
  - type: vpc_association
    content: "<VPC association status>"
severity: HIGH
mitigation:
  immediate: "Fix VPC association or namespace configuration"
  long_term: "Use IaC for namespace provisioning with validated VPC settings"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Specify a valid VPC ID at creation | GREEN |
| Use unique namespace names | GREEN |
| Grant servicediscovery and route53 permissions | YELLOW |
| Associate additional VPCs | YELLOW |
| Enable DNS hostnames and DNS support on VPC | GREEN |
| Tag namespaces | GREEN |

## Escalation Conditions

- Fix requires modifying production namespace or VPC configurations
- Fix involves associating VPCs that serve production workloads

## Data Sensitivity

- **MEDIUM**: namespace configurations expose service discovery topology and VPC architecture
- Route 53 hosted zone associations reveal network boundaries

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If VPC association was added incorrectly, disassociate: `aws route53 disassociate-vpc-from-hosted-zone --hosted-zone-id <zone-id> --vpc VPCRegion=<region>,VPCId=<vpc-id>`
2. If namespace was created with wrong configuration, delete only if no services are registered
3. If DNS settings were changed on VPC, restore previous settings
4. Verify rollback by confirming namespace and VPC associations match expected state

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
