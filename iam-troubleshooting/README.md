# IAM Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IAM problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for IAM — access denied root cause analysis, policy evaluation logic, role trust chain debugging, cross-account access, permission boundaries, SCP interactions, federation issues, credential management, and service-linked roles.

### Activate When

- Access denied errors on AWS API calls
- Policy evaluation confusion (why is this denied/allowed?)
- Role assumption failures
- Cross-account access issues
- Permission boundary conflicts
- SCP restrictions blocking operations
- MFA enforcement problems
- Credential rotation or expiry issues
- SAML/OIDC federation failures
- Service-linked role issues

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Access Denied** | A1–A4 | Policy evaluation, explicit deny, implicit deny, resource policy |
| **B — Role Issues** | B1–B4 | Trust policy, cross-account assume, session duration, chained roles |
| **C — Policy Management** | C1–C3 | Size limits, versioning, managed vs inline |
| **D — Permission Boundaries** | D1–D2 | Boundary conflicts, effective permissions |
| **E — SCP Interactions** | E1–E3 | SCP deny, evaluation, management account |
| **F — Federation** | F1–F3 | SAML, OIDC, web identity |
| **G — Credentials** | G1–G3 | Access key rotation, temporary credentials, MFA |
| **H — Cross-Account** | H1–H3 | Cross-account roles, resource sharing, confused deputy |
| **I — Service Roles** | I1–I2 | Service-linked roles, service roles |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
