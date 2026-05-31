# Amazon QLDB Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon QLDB problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon QLDB — ledgers, tables, PartiQL queries, journal export, verification, streaming, and performance.

### Activate When

- Ledger creation or configuration failures
- Table creation or index issues
- PartiQL query errors or syntax problems
- Journal export failures
- Cryptographic verification failures
- Kinesis streaming issues
- Performance or latency problems
- General QLDB errors

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Ledger** | A1–A2 | Creation failures, configuration |
| **B — Tables** | B1–B2 | Table operations, indexes |
| **C — PartiQL** | C1–C2 | Query errors, syntax issues |
| **D — Journal** | D1–D2 | Export failures, S3 output |
| **E — Verification** | E1–E2 | Digest verification, revision proof |
| **F — Streaming** | F1–F2 | Kinesis streaming, delivery issues |
| **G — Performance** | G1–G2 | Latency, throughput |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with QLDB permissions
- Permissions: `qldb:Describe*`, `qldb:List*`, `qldb:Get*`, `qldb:SendCommand`

---

## License

MIT-0
