# Cognito Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Cognito problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Cognito when the console alone isn't enough — user pool authentication flows, identity pool federation, token issues, Lambda trigger debugging, hosted UI problems, SAML/OIDC federation, MFA configuration, custom domains, app client configuration, and user migration.

### Activate When

- Sign-up or sign-in failures
- Token refresh or expiry issues
- Custom authentication flow errors
- SAML or OIDC federation problems
- Identity pool role mapping failures
- Lambda trigger timeouts or errors
- Hosted UI not loading or misconfigured
- MFA setup or verification failures
- App client configuration issues
- App client secret used in SPA/mobile
- User migration or import failures
- Custom domain certificate issues
- Adaptive authentication false positives
- Compromised credentials detection issues

---

## Skill Structure

```
cognito-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-sign-up-failures.md
    ├── A2-sign-in-failures.md
    ├── A3-token-refresh.md
    ├── A4-custom-auth-flow.md
    ├── B1-saml-federation.md
    ├── B2-oidc-federation.md
    ├── B3-identity-pool.md
    ├── C1-pre-sign-up-trigger.md
    ├── C2-pre-authentication-trigger.md
    ├── C3-post-confirmation-trigger.md
    ├── D1-sms-mfa.md
    ├── D2-totp-mfa.md
    ├── E1-hosted-ui-errors.md
    ├── E2-custom-domain.md
    ├── F1-client-config.md
    ├── F2-client-secret-issues.md
    ├── G1-user-migration.md
    ├── G2-user-import.md
    ├── H1-adaptive-auth.md
    ├── H2-compromised-credentials.md
    ├── Z1-general-troubleshooting.md
    ├── cognito-guardrails.md
    └── cognito-hallucination-patterns.yaml
```

---

## Runbook Library (28 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Auth Flows** | A1–A4 | Sign-up failures, sign-in failures, token refresh, custom auth |
| **B — Federation** | B1–B3 | SAML, OIDC, identity pool |
| **C — Triggers** | C1–C3 | Pre-sign-up, pre-authentication, post-confirmation |
| **D — MFA** | D1–D2 | SMS MFA, TOTP MFA |
| **E — Hosted UI** | E1–E2 | UI errors, custom domain |
| **F — App Client** | F1–F2 | Client config, secret issues |
| **G — User Management** | G1–G2 | Migration, import |
| **H — Advanced** | H1–H2 | Adaptive auth, compromised credentials |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
