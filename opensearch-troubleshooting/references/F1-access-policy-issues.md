---
title: "F1 — OpenSearch Access Policy Issues"
description: "Diagnose and resolve domain access policy denials and misconfigurations"
status: active
severity: HIGH
triggers:
  - "access denied"
  - "403 forbidden"
  - "access policy"
  - "resource policy"
  - "unauthorized"
  - "not authorized"
owner: devops-agent
objective: "Identify access policy misconfigurations and restore authorized access"
context: "OpenSearch domain access policies are resource-based JSON policies (similar to S3 bucket policies). They control who can access the domain endpoint and which OpenSearch actions are allowed. Access requires BOTH the resource policy AND the caller's IAM policy to allow the action. For VPC domains, IP-based conditions in the resource policy do not work — use security groups instead. FGAC adds another layer of authorization."
---

## Phase 1 — Triage

MUST:
- Check current access policy: `aws opensearch describe-domain-config --domain-name <domain> --query 'DomainConfig.AccessPolicies.Options' --output text | python3 -m json.tool`
- Check domain access type: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.{VPCOptions:VPCOptions,Endpoint:Endpoint,Endpoints:Endpoints}'`
- Check if FGAC is enabled: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.AdvancedSecurityOptions.{Enabled:Enabled,InternalUserDatabaseEnabled:InternalUserDatabaseEnabled}'`
- Test access: `curl -XGET "https://<endpoint>/_cluster/health" -v 2>&1 | grep -E "HTTP|403|401"`
- Check caller IAM permissions: `aws sts get-caller-identity`

SHOULD:
- Check CloudTrail for denied requests: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=es.amazonaws.com --max-results 20`
- Verify IAM policy allows es:ESHttp* actions
- Check for IP-based conditions in the access policy (only work for public domains)

MAY:
- Simulate IAM policy: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names es:ESHttpGet es:ESHttpPost --resource-arns arn:aws:es:<region>:<account>:domain/<domain>/*`
- Check for SCP restrictions in AWS Organizations

## Phase 2 — Remediate

MUST:
- If access policy too restrictive: update policy: `aws opensearch update-domain-config --domain-name <domain> --access-policies '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::<account>:role/<role>"},"Action":"es:*","Resource":"arn:aws:es:<region>:<account>:domain/<domain>/*"}]}'`
- If VPC domain with IP conditions: remove IP conditions and use security groups instead
- If FGAC blocking: check FGAC role mappings (see F2)

SHOULD:
- Use least-privilege access policies (specific actions, specific principals)
- For VPC domains, rely on security groups for network-level access control
- Document access policy changes in CloudTrail

MAY:
- Use tag-based access control for multi-tenant scenarios
- Implement condition keys for fine-grained resource policy control

## Common Issues

- symptoms: "403 Forbidden from Lambda function"
  diagnosis: "Lambda execution role not in the domain access policy."
  resolution: "Add Lambda role ARN to the access policy Principal."

- symptoms: "403 on VPC domain with IP-based policy"
  diagnosis: "IP conditions in resource policies don't work for VPC domains."
  resolution: "Remove IP conditions. Use security groups for VPC domain access control."

- symptoms: "Access works from CLI but not from application"
  diagnosis: "Application using different IAM role not in the access policy."
  resolution: "Add application's IAM role to the access policy."

## Output Format

```yaml
root_cause: "access_policy — <specific_cause>"
evidence:
  - type: access_policy
    content: "<current domain access policy>"
  - type: caller_identity
    content: "<IAM role/user making the request>"
  - type: domain_config
    content: "<VPC/public, FGAC enabled/disabled>"
severity: HIGH
mitigation:
  immediate: "Update access policy to allow authorized principals"
  long_term: "Implement least-privilege policies, document access requirements"
```


## Safety Ratings
```
safety_ratings:
  - "Check access policy and domain config: GREEN — read-only API calls"
  - "Test access: GREEN — read-only connectivity test"
  - "Update access policy: YELLOW — changes who can access the domain"
  - "Remove IP conditions on VPC domain: YELLOW — changes access control model"
```

## Escalation Conditions
- Domain serves production search
- Access policy changes affect multiple services and teams
- VPC domain with IP-based conditions (ineffective, needs restructuring)
- FGAC adding additional authorization layer
- SCP restrictions blocking access

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Access policies: account IDs and IAM role ARNs"
    - "Domain endpoint: connection details"
    - "FGAC configuration: internal authorization rules"
  handling: "Access policies contain account IDs and role ARNs. Do not expose externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER use wildcard (*) principals in production access policies
- NEVER remove access policy entirely — always maintain least-privilege access

## Phase 3 — Rollback
- If access policy was updated: restore previous policy JSON
- If IP conditions were removed: re-add if needed for public domains
- If FGAC role mappings were changed: revert using master user credentials

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
