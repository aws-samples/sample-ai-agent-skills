---
title: "I1 — OpenSearch Serverless Collection Issues"
description: "Diagnose and resolve OpenSearch Serverless collection creation, access, and operational issues"
status: active
severity: HIGH
triggers:
  - "Serverless collection"
  - "collection failed"
  - "collection access"
  - "data access policy"
  - "network policy"
  - "encryption policy"
owner: devops-agent
objective: "Resolve Serverless collection issues and restore normal operation"
context: "OpenSearch Serverless uses collections instead of domains. Collections require three types of policies: encryption policy (must exist before creation), network policy (controls public/VPC access), and data access policy (controls index-level permissions). Collection types are search, time-series, and vector search — type cannot be changed after creation. Serverless manages infrastructure automatically — no nodes, shards, or JVM to configure."
---

## Phase 1 — Triage

MUST:
- Check collection status: `aws opensearchserverless batch-get-collection --names <collection-name>`
- Check encryption policy: `aws opensearchserverless list-security-policies --type encryption`
- Check network policy: `aws opensearchserverless list-security-policies --type network`
- Check data access policy: `aws opensearchserverless list-access-policies --type data`
- Check collection endpoint: `aws opensearchserverless batch-get-collection --names <collection-name> --query 'collectionDetails[*].{Name:name,Status:status,Endpoint:collectionEndpoint,DashboardEndpoint:dashboardEndpoint}'`

SHOULD:
- Check if encryption policy matches collection name/pattern: `aws opensearchserverless get-security-policy --name <policy-name> --type encryption`
- Check network policy details: `aws opensearchserverless get-security-policy --name <policy-name> --type network`
- Check data access policy details: `aws opensearchserverless get-access-policy --name <policy-name> --type data`
- Check account-level limits: `aws opensearchserverless list-collections`

MAY:
- Check VPC endpoint if using VPC access: `aws opensearchserverless list-vpc-endpoints`
- Check CloudTrail for collection API calls
- Test collection access: `curl -XGET "https://<collection-endpoint>" --aws-sigv4 "aws:amz:us-east-1:aoss"`

## Phase 2 — Remediate

MUST:
- If collection creation fails: ensure encryption policy exists matching the collection name
- If access denied: update data access policy to grant permissions to the IAM principal
- If network unreachable: check network policy allows access from the client's network
- If VPC access needed: create VPC endpoint for OpenSearch Serverless

SHOULD:
- Create encryption policy before collection: `aws opensearchserverless create-security-policy --name <name> --type encryption --policy '{"Rules":[{"ResourceType":"collection","Resource":["collection/<name>"]}],"AWSOwnedKey":true}'`
- Create network policy: `aws opensearchserverless create-security-policy --name <name> --type network --policy '[{"Rules":[{"ResourceType":"collection","Resource":["collection/<name>"]}],"AllowFromPublic":true}]'`
- Create data access policy: `aws opensearchserverless create-access-policy --name <name> --type data --policy '[{"Rules":[{"ResourceType":"index","Resource":["index/<collection>/*"],"Permission":["aoss:*"]}],"Principal":["arn:aws:iam::<account>:role/<role>"]}]'`

MAY:
- Use SAML authentication for Dashboards access
- Set up lifecycle policies for time-series collections
- Monitor OCU usage via CloudWatch

## Common Issues

- symptoms: "Collection stuck in CREATING status"
  diagnosis: "Missing encryption policy matching the collection name."
  resolution: "Create encryption policy with resource pattern matching the collection name."

- symptoms: "403 Forbidden when accessing collection"
  diagnosis: "Data access policy does not grant permissions to the IAM principal."
  resolution: "Update data access policy to include the IAM role/user ARN."

- symptoms: "Cannot reach collection endpoint"
  diagnosis: "Network policy restricts access. VPC-only access configured but client is public."
  resolution: "Update network policy to allow public access or create VPC endpoint."

## Output Format

```yaml
root_cause: "serverless_collection — <specific_cause>"
evidence:
  - type: collection_status
    content: "<collection status and configuration>"
  - type: policies
    content: "<encryption, network, data access policies>"
  - type: access_test
    content: "<connection test result>"
severity: HIGH
mitigation:
  immediate: "Fix missing or misconfigured policies"
  long_term: "Document policy requirements, use infrastructure as code for policy management"
```


## Safety Ratings
```
safety_ratings:
  - "Check collection status and policies: GREEN — read-only API calls"
  - "Create encryption policy: GREEN — required for collection creation"
  - "Create network policy: YELLOW — controls collection access"
  - "Update data access policy: YELLOW — changes who can access collection data"
  - "Create VPC endpoint: YELLOW — creates network path to collection"
  - "Delete collection: RED — permanently removes collection and data"
```

## Escalation Conditions
- Domain serves production search
- Collection stuck in CREATING due to missing policies
- Data access policy changes affecting multiple teams
- Network policy changes affecting collection accessibility
- VPC endpoint needed for private access

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Collection data: indexed business content"
    - "Data access policies: IAM principal access configuration"
    - "Network policies: access control configuration"
    - "Encryption policies: encryption configuration"
  handling: "Data access policies contain IAM principal ARNs. Do not expose externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER delete a collection without confirming data is backed up or no longer needed
- NEVER use AllowFromPublic in network policies for sensitive collections
- NEVER grant aoss:* to untrusted principals in data access policies

## Phase 3 — Rollback
- If data access policy was updated: restore previous policy
- If network policy was changed: revert to previous network policy
- If VPC endpoint was created: delete the endpoint if not needed
- If collection was deleted: CANNOT be recovered — data is permanently lost

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling fine-grained access control"
  - "NEVER suggest public access domains"
  - "NEVER suggest disabling encryption at rest"
