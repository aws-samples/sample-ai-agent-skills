# Amazon Verified Permissions Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Verified Permissions problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Verified Permissions — policy store creation and configuration, Cedar policy authoring and validation, schema definition and entity types, authorization decision debugging, identity source integration with Cognito/OIDC, policy templates and template-linked policies, and batch authorization.

### Activate When

- Policy store creation or configuration failures
- Cedar policy syntax or validation errors
- Schema definition or entity type issues
- Authorization decisions returning unexpected DENY
- Identity source (Cognito/OIDC) integration problems
- Policy template creation or linking failures
- Batch authorization errors or throttling
- General Verified Permissions errors without clear symptoms

---

## Skill Structure

```
verifiedpermissions-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-policy-store-creation.md
    ├── A2-policy-store-config.md
    ├── B1-cedar-policy-authoring.md
    ├── B2-policy-validation.md
    ├── C1-schema-definition.md
    ├── C2-schema-validation.md
    ├── D1-authorization-decisions.md
    ├── D2-token-authorization.md
    ├── E1-cognito-identity-source.md
    ├── E2-oidc-identity-source.md
    ├── F1-policy-templates.md
    ├── F2-template-linked-policies.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Policy Store** | A1–A2 | Store creation, store configuration |
| **B — Cedar Policies** | B1–B2 | Policy authoring, policy validation |
| **C — Schema** | C1–C2 | Schema definition, schema validation |
| **D — Authorization** | D1–D2 | Decision debugging, token-based auth |
| **E — Identity Source** | E1–E2 | Cognito integration, OIDC setup |
| **F — Templates** | F1–F2 | Template creation, template-linked policies |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `verifiedpermissions:*`, `cognito-idp:Describe*`, `cloudtrail:LookupEvents`
- Cognito user pool access for identity source troubleshooting

---

## License

MIT-0
