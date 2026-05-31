---
title: "D1 — Redshift Query Access"
description: "Diagnose Redshift data share access issues via Data Exchange"
status: active
severity: HIGH
triggers: ["Redshift data share error", "cannot query shared data", "datashare not found", "cross-account Redshift access"]
owner: devops-agent
objective: "Resolve Redshift data share access issues including RAM invitations, cluster configuration, and query permissions"
context: "Redshift data shares in Data Exchange use AWS RAM for cross-account sharing. Subscribers must accept RAM invitations and associate the data share with their Redshift cluster."
---

## Phase 1 — Triage

MUST:
- Check RAM resource share invitations: `aws ram get-resource-share-invitations --query 'resourceShareInvitations[?status==`PENDING`]'`
- Verify data share association: `aws redshift describe-data-shares --data-share-arn <arn>`
- Check Redshift cluster status: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].{Status:ClusterStatus,Endpoint:Endpoint}'`

SHOULD:
- Verify the data share is associated with the subscriber's Redshift cluster
- Check Redshift database permissions for the querying user
- Verify the Redshift cluster is in the same region as the data share

MAY:
- Test query access: `SELECT * FROM datashare_schema.table LIMIT 1;`
- Check Redshift audit logs for access denied events

## Phase 2 — Remediate

MUST:
- Accept pending RAM resource share invitation: `aws ram accept-resource-share-invitation --resource-share-invitation-arn <arn>`
- Associate data share with Redshift cluster: `aws redshift associate-data-share-consumer --data-share-arn <arn> --consumer-identifier <cluster-namespace>`
- Create external database from data share in Redshift

SHOULD:
- Grant SELECT permissions on shared schemas and tables to querying users
- Verify Redshift cluster has network access for cross-account data share

MAY:
- Set up Redshift Spectrum for querying shared data alongside local data
- Configure query monitoring rules for shared data access

## Safety Ratings

```
safety_ratings:
  - "ram get-resource-share-invitations: GREEN — read-only RAM check"
  - "redshift describe-data-shares: GREEN — read-only data share status"
  - "ram accept-resource-share-invitation: YELLOW — accepts cross-account resource sharing"
  - "redshift associate-data-share-consumer: YELLOW — associates data share with cluster"
```

## Escalation Conditions

- RAM invitation expired and provider must re-send
- Redshift cluster version does not support data sharing
- Cross-account data share requires VPC peering or PrivateLink
- Data share contains tables with unsupported data types

## Data Sensitivity

- `aws ram get-resource-share-invitations`: MEDIUM — reveals cross-account sharing details
- `aws redshift describe-data-shares`: MEDIUM — contains data share ARN and consumer details
- `aws redshift describe-clusters`: MEDIUM — contains cluster endpoint and configuration
- `Redshift query results`: HIGH — contains actual shared data

## Prohibited Actions

- NEVER accept RAM invitations from unknown or untrusted accounts
- NEVER grant public access to databases created from data shares
- NEVER modify the provider's data share configuration from the subscriber side
- NEVER query shared data without complying with the provider's data usage terms

## Phase 3 — Rollback

1. If data share was associated with wrong cluster, disassociate: `aws redshift disassociate-data-share-consumer --data-share-arn <arn>`
2. If RAM invitation was accepted incorrectly, contact provider to revoke
3. Drop external database if created from wrong data share
4. Verify no unauthorized data access occurred
5. Re-associate with correct cluster if needed

## Common Issues

- symptoms: "Cannot query data share — database not found"
  diagnosis: "RAM invitation not accepted or data share not associated with cluster."
  resolution: "Accept RAM invitation, associate data share, create external database."

## Output Format

```yaml
root_cause: "redshift_query_access — <specific_cause>"
evidence:
  - type: redshift_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix data share access"
  long_term: "Document data share onboarding procedure"
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
