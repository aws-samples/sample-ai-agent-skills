# Amazon WorkSpaces Web Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon WorkSpaces Web (WorkSpaces Secure Browser) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon WorkSpaces Web — portal creation, configuration, browser sessions, connectivity, identity provider setup, SAML authentication, network settings, VPC configuration, browser policies, user access logging, and IP access settings.

### Activate When

- Portal creation failures
- Portal configuration issues
- Browser session errors
- Session connectivity problems
- Identity provider configuration issues
- SAML authentication failures
- Network settings problems
- VPC configuration errors
- Browser policy issues
- User access logging gaps
- IP access restriction problems
- General WorkSpaces Web errors

---

## Skill Structure

```
workspaces-web-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-portal-creation-failures.md
    ├── A2-portal-configuration.md
    ├── B1-browser-session-errors.md
    ├── B2-session-connectivity-issues.md
    ├── C1-identity-provider-configuration.md
    ├── C2-saml-authentication-errors.md
    ├── D1-network-settings.md
    ├── D2-vpc-configuration.md
    ├── E1-browser-policy-issues.md
    ├── E2-user-access-logging.md
    ├── F1-ip-access-settings.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Portal** | A1–A2 | Creation failures, configuration |
| **B — Sessions** | B1–B2 | Session errors, connectivity |
| **C — Identity** | C1–C2 | IdP configuration, SAML auth |
| **D — Network** | D1–D2 | Network settings, VPC config |
| **E — Policy** | E1–E2 | Browser policies, access logging |
| **F — Access** | F1 | IP access settings |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `workspaces-web:*`, `ec2:Describe*`, `iam:Get*`, `kms:Describe*`, `logs:*` permissions
- Identity provider admin access for SAML troubleshooting

---

## License

MIT-0
