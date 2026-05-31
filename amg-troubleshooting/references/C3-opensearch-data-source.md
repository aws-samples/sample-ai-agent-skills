---
title: "C3 — OpenSearch Data Source Issues"
description: "Diagnose OpenSearch data source connectivity and query problems in Amazon Managed Grafana"
status: active
severity: HIGH
triggers:
  - "OpenSearch data source"
  - "Elasticsearch data source"
  - "OpenSearch query failed"
  - "OpenSearch access denied"
  - "OpenSearch connection"
owner: devops-agent
objective: "Identify why OpenSearch data source queries fail in AMG and restore log/search access"
context: "AMG can connect to Amazon OpenSearch Service as a data source for log analytics and search. It requires the workspace IAM role to have OpenSearch permissions. If the OpenSearch domain is in a VPC, the AMG workspace must have VPC configuration. Issues typically involve IAM permissions, VPC connectivity, or fine-grained access control."
---

## Phase 1 — Triage

MUST:
- Verify AMAZON_OPENSEARCH_SERVICE is enabled as a data source type: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.dataSources'`
- Check workspace IAM role for OpenSearch permissions: `aws iam list-attached-role-policies --role-name <workspace-role-name>`
- Get OpenSearch domain details: `aws opensearch describe-domain --domain-name <domain-name>`
- Check if the OpenSearch domain is in a VPC: `aws opensearch describe-domain --domain-name <domain-name> --query 'DomainStatus.VPCOptions'`

SHOULD:
- Verify the workspace has VPC configuration if OpenSearch is in a VPC: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.vpcConfiguration'`
- Check OpenSearch domain access policies: `aws opensearch describe-domain --domain-name <domain-name> --query 'DomainStatus.AccessPolicies'`
- Verify fine-grained access control settings if enabled

MAY:
- Check security group rules allow traffic between workspace and OpenSearch domain
- Test OpenSearch connectivity from within the VPC

## Phase 2 — Remediate

MUST:
- Add OpenSearch read permissions to the workspace IAM role (es:ESHttpGet, es:ESHttpPost, es:ESHttpHead)
- Configure VPC connectivity on the workspace if OpenSearch is in a VPC
- Ensure security groups allow HTTPS (443) traffic from workspace to OpenSearch

SHOULD:
- Configure SigV4 authentication on the OpenSearch data source in Grafana
- Map the workspace IAM role in OpenSearch fine-grained access control if enabled
- Set the correct index pattern in the data source configuration

MAY:
- Configure multiple OpenSearch data sources for different domains or indices
- Set up index lifecycle policies for log retention

## Common Issues

- symptoms: "OpenSearch data source test fails with connection timeout"
  diagnosis: "OpenSearch domain is in a VPC but the AMG workspace lacks VPC configuration."
  resolution: "Configure VPC connectivity on the workspace with subnets and security groups that can reach the OpenSearch domain."

- symptoms: "OpenSearch data source returns 403 Forbidden"
  diagnosis: "Workspace IAM role lacks OpenSearch permissions, or fine-grained access control blocks the role."
  resolution: "Add es:ESHttp* permissions to the workspace role. Map the role in OpenSearch fine-grained access control."

- symptoms: "Queries return no results but data exists in OpenSearch"
  diagnosis: "Index pattern in the data source does not match actual index names, or time field is misconfigured."
  resolution: "Verify the index pattern and time field name match the OpenSearch index configuration."

## Safety Ratings

```
safety_ratings:
  - "describe-domain: GREEN — read-only OpenSearch domain check"
  - "describe-workspace (vpcConfiguration): GREEN — read-only VPC check"
  - "add es:ESHttp* permissions: YELLOW — grants OpenSearch read/write access"
  - "configure VPC on workspace: YELLOW — creates ENIs, changes network topology"
  - "map role in fine-grained access control: YELLOW — changes OpenSearch authorization"
```

## Escalation Conditions

- OpenSearch domain is in a VPC and workspace VPC configuration requires network team involvement
- Fine-grained access control changes require OpenSearch domain administrator access
- OpenSearch domain access policy changes affect other applications
- VPC connectivity issues affect multiple data sources, not just OpenSearch

## Data Sensitivity

- `aws opensearch describe-domain`: MEDIUM — reveals domain configuration and VPC details
- OpenSearch data: HIGH — may contain application logs, search indices, and sensitive data
- Fine-grained access control mappings: MEDIUM — reveals authorization configuration

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER grant OpenSearch write permissions unless explicitly required for annotation storage
- NEVER modify OpenSearch domain access policies without coordinating with other consumers

## Phase 3 — Rollback

1. If IAM policies were changed, detach incorrect policies: `aws iam detach-role-policy --role-name <role> --policy-arn <policy-arn>`
2. If VPC configuration was added to the workspace incorrectly, update it: `aws grafana update-workspace --workspace-id <id> --remove-vpc-configuration`
3. If fine-grained access control mappings were changed, restore previous role mappings in OpenSearch
4. Verify OpenSearch data source connectivity after rollback
5. Confirm other applications using the OpenSearch domain are unaffected

## Output Format

```yaml
root_cause: "opensearch_data_source — <specific_cause>"
evidence:
  - type: data_source_config
    content: "<OpenSearch data source configuration>"
  - type: iam_role
    content: "<workspace IAM role OpenSearch permissions>"
  - type: opensearch_domain
    content: "<domain VPC and access policy configuration>"
severity: HIGH
mitigation:
  immediate: "Fix VPC connectivity, IAM permissions, or fine-grained access control"
  long_term: "Document OpenSearch access requirements and automate VPC configuration"
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
