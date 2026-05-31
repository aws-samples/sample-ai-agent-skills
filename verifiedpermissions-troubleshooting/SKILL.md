---
name: verifiedpermissions-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Verified Permissions
  problems by analyzing policy stores, Cedar policies, schemas, authorization
  decisions, identity sources, policy templates, batch authorization, and
  following structured runbooks. Activate when: policy store creation failures,
  Cedar policy validation errors, schema definition issues, authorization
  decision debugging, identity source configuration, policy template problems,
  batch authorization failures, or the user says something is wrong with
  Verified Permissions without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with verifiedpermissions:*, IAM, CloudTrail,
  and optionally Cognito permissions for identity source integration.
---

# Amazon Verified Permissions Diagnostics

## When to use

Any Amazon Verified Permissions investigation — policy store setup, Cedar policy authoring, schema validation, authorization decision debugging, identity source integration, policy template management, or batch authorization issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws verifiedpermissions list-policy-stores
aws verifiedpermissions get-policy-store --policy-store-id <id>
aws verifiedpermissions get-schema --policy-store-id <id>
aws verifiedpermissions list-policies --policy-store-id <id>
aws verifiedpermissions list-identity-sources --policy-store-id <id>
```

### Step 2 — Domain deep dive

```
aws verifiedpermissions get-policy --policy-store-id <id> --policy-id <policy-id>
aws verifiedpermissions is-authorized --policy-store-id <id> --principal <json> --action <json> --resource <json>
aws verifiedpermissions list-policy-templates --policy-store-id <id>
```

### Step 3 — Detailed investigation

```
aws verifiedpermissions is-authorized-with-token --policy-store-id <id> --identity-token <token> --action <json> --resource <json>
aws verifiedpermissions batch-is-authorized --policy-store-id <id> --requests <json>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=verifiedpermissions.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Verified Permissions issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-policy-stores` | Enumerate policy stores |
| `get-schema` | Check entity types and actions |
| `list-policies` | List all policies in a store |
| `is-authorized` | Test authorization decisions |
| `is-authorized-with-token` | Test with Cognito/OIDC token |
| `batch-is-authorized` | Test multiple decisions at once |
| `list-policy-templates` | Check reusable templates |
| CloudTrail | Audit policy and authorization changes |

## Gotchas: Amazon Verified Permissions

- Verified Permissions uses Cedar policy language, NOT IAM JSON. Cedar has its own syntax with `permit`/`forbid` and `when`/`unless` clauses.
- Schema validation is optional but strongly recommended. Without a schema, Cedar policies are not validated against entity types and actions.
- Identity sources connect Cognito user pools or OIDC providers. Token claims are mapped to Cedar entity attributes automatically.
- Policy templates use placeholders (`?principal`, `?resource`) for reusable policies. Template-linked policies fill in the placeholders.
- `is-authorized` returns ALLOW only if at least one `permit` matches AND no `forbid` matches. Explicit deny always wins.
- Batch authorization (`batch-is-authorized`) has a limit of 30 requests per call.

## Anti-hallucination rules

1. Always cite specific policy store IDs, policy IDs, or API responses as evidence.
2. Cedar is NOT IAM JSON. Never suggest IAM policy syntax for Verified Permissions.
3. Schema defines entity types and actions — it is NOT an IAM resource policy.
4. `is-authorized` evaluates Cedar policies, NOT IAM policies.
5. Policy templates are NOT IAM managed policies.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## Runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Policy Store | A1–A2 | Store creation, store configuration |
| B — Cedar Policies | B1–B2 | Policy authoring, policy validation |
| C — Schema | C1–C2 | Schema definition, schema validation |
| D — Authorization | D1–D2 | Decision debugging, token-based auth |
| E — Identity Source | E1–E2 | Cognito integration, OIDC setup |
| F — Templates | F1–F2 | Template creation, template-linked policies |
| Z — Catch-All | Z1 | General troubleshooting |
