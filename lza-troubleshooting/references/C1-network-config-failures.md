---
title: "C1 — Network Config Failures"
description: "Diagnose and resolve network-config.yaml configuration and deployment failures"
status: active
severity: HIGH
triggers: ["network config error", "VPC creation failed", "transit gateway error", "network-config.yaml invalid", "CIDR conflict"]
owner: devops-agent
objective: "Fix network-config.yaml errors to enable successful network infrastructure deployment"
context: "The network-config.yaml is the most complex LZA configuration file. It defines VPCs, subnets, transit gateways, route tables, security groups, NACLs, VPN connections, Direct Connect, and more. Network configuration errors are the most common cause of LZA deployment failures."
---

## Phase 1 — Triage

MUST:
- Download and review network-config.yaml: `aws s3 cp s3://aws-accelerator-config-<account-id>-<region>/network-config.yaml ./`
- Check CloudFormation stack events for network stacks: `aws cloudformation describe-stack-events --stack-name AWSAccelerator-NetworkVpcStack-<account>-<region> --max-items 20`
- Verify CIDR ranges do not overlap across VPCs and subnets

SHOULD:
- Check existing VPCs in target accounts: `aws ec2 describe-vpcs --query 'Vpcs[*].{Id:VpcId,Cidr:CidrBlock,Name:Tags[?Key==`Name`].Value|[0]}'`
- Verify transit gateway exists if referenced: `aws ec2 describe-transit-gateways`
- Review CodeBuild logs for the Network stage

MAY:
- Check for subnet CIDR calculation errors
- Verify availability zone names match the target region
- Check for route table circular references

## Phase 2 — Remediate

MUST:
- Fix CIDR overlaps between VPCs and subnets
- Correct transit gateway references and route table configurations
- Ensure availability zone names are valid for the target region

SHOULD:
- Validate subnet CIDRs fit within VPC CIDR ranges
- Verify security group rules reference valid CIDR ranges or security group IDs
- Check NACL rules for correct ordering and protocol specifications

MAY:
- Use a CIDR calculator to plan non-overlapping address spaces
- Implement network configuration validation scripts

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-vpcs: GREEN — read-only VPC listing"
  - "ec2 describe-transit-gateways: GREEN — read-only TGW listing"
  - "cloudformation describe-stack-events: GREEN — read-only event history"
  - "Network config deployment: RED — creates/modifies network infrastructure across accounts"
```

## Escalation Conditions

- CIDR conflicts with existing non-LZA VPCs that cannot be changed
- Transit gateway attachment limits reached
- Network changes would disrupt production connectivity
- Cross-region or cross-account networking requires additional permissions

## Data Sensitivity

- network-config.yaml: HIGH — reveals complete network architecture
- VPC and subnet details: MEDIUM — reveals network topology
- Security group rules: MEDIUM — reveals network access patterns
- Transit gateway configuration: HIGH — reveals inter-account connectivity

## Prohibited Actions

- NEVER deploy network changes without reviewing the impact on existing connectivity
- NEVER delete VPCs or subnets that have running workloads
- NEVER modify transit gateway routing without understanding the full routing topology
- NEVER use overlapping CIDR ranges between VPCs connected via transit gateway

## Phase 3 — Rollback

1. If network changes disrupted connectivity, revert network-config.yaml and re-run the pipeline
2. If VPCs were created with wrong CIDRs, they may need to be deleted and recreated (disruptive)
3. If transit gateway routes were incorrect, update the configuration and re-deploy
4. Document all network changes and their impact on connectivity
5. Verify connectivity between all accounts and regions after changes

## Common Issues

- symptoms: "CloudFormation fails with 'The CIDR block conflicts with another subnet'"
  diagnosis: "Subnet CIDR ranges overlap within the same VPC or with existing subnets."
  resolution: "Review all subnet CIDRs in the VPC. Ensure no overlaps. Use a CIDR calculator."

- symptoms: "Transit gateway attachment fails"
  diagnosis: "Transit gateway does not exist, attachment limit reached, or VPC is in wrong account."
  resolution: "Verify TGW exists. Check attachment limits. Ensure VPC and TGW are in the correct accounts."

## Output Format

```yaml
root_cause: "network_config — <specific_cause>"
evidence:
  - type: network_error
    content: "<CIDR details, stack events, and configuration>"
severity: HIGH
mitigation:
  immediate: "Fix network-config.yaml"
  long_term: "Implement CIDR planning and network validation"
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
