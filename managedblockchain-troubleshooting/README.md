# Amazon Managed Blockchain Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Managed Blockchain problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Managed Blockchain — network creation, member/node management, Hyperledger Fabric, Ethereum, chaincode, and peer connectivity.

### Activate When

- Network creation or governance failures
- Member or node management issues
- Hyperledger Fabric chaincode errors
- Fabric channel configuration problems
- Ethereum node sync issues
- JSON-RPC connectivity failures
- VPC endpoint or peer connectivity problems
- General Managed Blockchain errors

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Network** | A1–A2 | Creation failures, governance |
| **B — Members/Nodes** | B1–B3 | Member issues, node management, scaling |
| **C — Fabric** | C1–C3 | Chaincode, channels, endorsement |
| **D — Ethereum** | D1–D2 | Node sync, JSON-RPC |
| **E — Connectivity** | E1–E2 | VPC endpoints, peer connectivity |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with ManagedBlockchain permissions
- Permissions: `managedblockchain:Get*`, `managedblockchain:List*`

---

## License

MIT-0
