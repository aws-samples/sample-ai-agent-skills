# AWS HealthLake Diagnostics Skill

Agent skill for investigating and troubleshooting AWS HealthLake problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS HealthLake — data stores, FHIR import/export, search, analytics, NLP enrichment, and security.

### Activate When

- Data store creation or configuration failures
- FHIR import job errors or data validation failures
- Export job failures or S3 output issues
- Search query errors or performance problems
- Analytics integration issues
- NLP enrichment not working
- Encryption or access control problems
- General HealthLake errors

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Data Store** | A1–A2 | Creation failures, configuration |
| **B — Import** | B1–B2 | FHIR import errors, data validation |
| **C — Export** | C1–C2 | Export failures, S3 output |
| **D — Search** | D1–D2 | Search queries, performance |
| **E — Analytics** | E1–E2 | Analytics queries, Lake Formation |
| **F — NLP** | F1–F2 | NLP enrichment, entity extraction |
| **G — Security** | G1–G2 | Encryption, access control |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with HealthLake permissions
- Permissions: `healthlake:Describe*`, `healthlake:List*`, `healthlake:Read*`, `healthlake:Search*`

---

## License

MIT-0
