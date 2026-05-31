# ACM Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Certificate Manager (ACM) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for ACM when the console alone isn't enough — certificate validation failures, DNS and email validation issues, auto-renewal problems, imported certificate expiry, CloudFront us-east-1 requirement, ALB/NLB certificate binding, API Gateway custom domains, private CA management, certificate import and chain errors, key algorithm compatibility, certificate transparency, and deletion issues.

### Activate When

- DNS validation CNAME not propagating
- Email validation not received or expired
- Certificate stuck in PENDING_VALIDATION
- Auto-renewal failure notifications
- Imported certificate approaching expiry
- CloudFront distribution rejecting certificate (wrong region)
- ALB/NLB listener certificate errors
- API Gateway custom domain certificate issues
- Certificate not appearing in service dropdown
- Private CA creation or configuration failures
- Private certificate issuance errors
- Certificate import failures (format, chain, key mismatch)
- Chain of trust validation errors
- Key algorithm incompatibility with AWS services
- Certificate transparency logging questions
- Cannot delete certificate (in-use error)
- Orphaned or unused certificates
- Wildcard certificate not covering expected domains

---

## Skill Structure

```
acm-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-dns-validation-failures.md
    ├── A2-email-validation.md
    ├── A3-validation-pending.md
    ├── B1-auto-renewal-failures.md
    ├── B2-manual-renewal.md
    ├── B3-imported-cert-expiry.md
    ├── C1-cloudfront-us-east-1.md
    ├── C2-alb-nlb-binding.md
    ├── C3-api-gateway-custom-domain.md
    ├── C4-cert-not-showing-in-service.md
    ├── D1-private-ca-issues.md
    ├── D2-private-cert-issuance.md
    ├── E1-import-failures.md
    ├── E2-chain-issues.md
    ├── F1-key-algorithm.md
    ├── F2-certificate-transparency.md
    ├── G1-in-use-certificate.md
    ├── G2-orphaned-certificates.md
    ├── Z1-general-troubleshooting.md
    ├── acm-guardrails.md
    └── acm-hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Validation** | A1–A3 | DNS validation failures, email validation, validation pending/timeout |
| **B — Renewal** | B1–B3 | Auto-renewal failures, manual renewal, imported cert expiry |
| **C — Integration** | C1–C4 | CloudFront (us-east-1), ALB/NLB, API Gateway, cert not showing |
| **D — Private CA** | D1–D2 | Private CA issues, private cert issuance |
| **E — Import** | E1–E2 | Import failures, chain of trust issues |
| **F — Security** | F1–F2 | Key algorithm requirements, certificate transparency |
| **G — Deletion** | G1–G2 | In-use certificate, orphaned certificates |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
