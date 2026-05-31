---
title: "G2 — Private Data Source Access Issues"
description: "Diagnose connectivity to private data sources from Amazon Managed Grafana"
status: active
severity: HIGH
triggers:
  - "private data source"
  - "private endpoint"
  - "internal data source"
  - "VPC data source"
  - "cannot connect to data source"
owner: devops-agent
objective: "Identify why the AMG workspace cannot reach private data sources and establish connectivity"
context: "Private data sources (self-managed Prometheus, Elasticsearch, databases, custom APIs) in a VPC require the AMG workspace to have VPC configuration. The workspace creates ENIs in the specified subnets to reach private resources. DNS resolution, security groups, and network routing must all be correctly configured."
---

## Phase 1 — Triage

MUST:
- Verify the workspace has VPC configuration: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.vpcConfiguration'`
- Check if the data source endpoint is a private IP or private DNS name
- Verify the workspace subnets can route to the data source's subnet
- Check security groups on both the workspace and the data source

SHOULD:
- Verify DNS resolution for the data source endpoint within the VPC
- Check if the data source requires specific ports beyond HTTPS (443)
- Verify the data source is running and accepting connections

MAY:
- Check VPC peering or Transit Gateway if the data source is in a different VPC
- Verify TLS/SSL certificate configuration if the data source uses HTTPS
- Check if a proxy is needed between the workspace and the data source

## Phase 2 — Remediate

MUST:
- Configure VPC connectivity on the workspace if not already set
- Ensure network path exists between workspace subnets and data source subnets
- Configure security groups to allow traffic on the required ports

SHOULD:
- Use private DNS names with Route 53 private hosted zones for service discovery
- Configure VPC endpoints for AWS services used as data sources
- Test connectivity with a simple health check before configuring complex queries

MAY:
- Set up VPC peering for cross-VPC data source access
- Implement a proxy or load balancer for data sources that require connection pooling

## Common Issues

- symptoms: "Data source test fails with 'connection refused' or timeout"
  diagnosis: "Workspace does not have VPC configuration, or security groups block the connection."
  resolution: "Configure VPC on the workspace. Ensure security groups allow traffic on the data source port."

- symptoms: "Data source works intermittently"
  diagnosis: "Workspace ENIs are in subnets that don't have consistent routing to the data source, or the data source has availability issues."
  resolution: "Verify subnet routing. Ensure the data source is highly available across the same AZs as the workspace subnets."

- symptoms: "Private DNS name does not resolve from the workspace"
  diagnosis: "VPC DNS resolution is not enabled, or the private hosted zone is not associated with the workspace VPC."
  resolution: "Enable DNS resolution and DNS hostnames on the VPC. Associate the private hosted zone with the VPC."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace (vpcConfiguration): GREEN — read-only check"
  - "test data source connectivity: GREEN — read-only health check"
  - "configure VPC on workspace: YELLOW — creates ENIs, changes network topology"
  - "modify security groups for data source access: YELLOW — changes network access controls"
  - "set up VPC peering: RED — establishes cross-VPC network connectivity"
```

## Escalation Conditions

- Private data source connectivity affects production monitoring
- VPC peering or Transit Gateway required for cross-VPC access
- DNS resolution issues require Route 53 private hosted zone changes
- Security group changes affect other applications in the VPC

## Data Sensitivity

- Data source endpoints: MEDIUM — reveals internal service locations
- VPC configuration: MEDIUM — reveals network architecture
- Security group rules: MEDIUM — reveals network access controls
- Private DNS records: MEDIUM — reveals internal service discovery

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER expose private data sources to the public internet for troubleshooting
- NEVER modify security groups on the data source without coordinating with the data source owner

## Phase 3 — Rollback

1. If VPC configuration was added incorrectly, remove it: `aws grafana update-workspace --workspace-id <id> --remove-vpc-configuration`
2. If security group rules were added, remove the rules that were added for workspace access
3. If VPC peering was established, delete the peering connection if no longer needed
4. If Route 53 private hosted zone was modified, restore original records
5. Verify the data source is still accessible to its other consumers after rollback

## Output Format

```yaml
root_cause: "private_data_source — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<workspace VPC configuration>"
  - type: network_path
    content: "<routing and security group analysis>"
  - type: data_source_endpoint
    content: "<data source endpoint and port>"
severity: HIGH
mitigation:
  immediate: "Establish network connectivity between workspace and data source"
  long_term: "Document private data source network requirements and automate with IaC"
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
