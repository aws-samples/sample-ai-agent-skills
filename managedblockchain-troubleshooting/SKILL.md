---
name: managedblockchain-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Managed Blockchain
  problems by analyzing network creation, member/node management,
  Hyperledger Fabric, Ethereum, chaincode, and peer connectivity using
  structured runbooks. Activate when: network creation failures, member
  or node issues, Fabric chaincode errors, Ethereum node problems, peer
  connectivity failures, or the user says something is wrong with
  Managed Blockchain.
compatibility: >
  Requires AWS CLI or SDK access with ManagedBlockchain, IAM, CloudWatch,
  and VPC permissions as applicable.
---

# Amazon Managed Blockchain Diagnostics

## When to use

Any Amazon Managed Blockchain investigation — network creation, member/node management, Hyperledger Fabric, Ethereum, chaincode, or peer connectivity troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws managedblockchain list-networks
aws managedblockchain get-network --network-id <id>
aws managedblockchain list-members --network-id <id>
aws managedblockchain get-member --network-id <id> --member-id <member-id>
```

### Step 2 — Domain deep dive

```
aws managedblockchain list-nodes --network-id <id> --member-id <member-id>
aws managedblockchain get-node --network-id <id> --member-id <member-id> --node-id <node-id>
aws managedblockchain list-proposals --network-id <id>
aws managedblockchain get-proposal --network-id <id> --proposal-id <proposal-id>
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/ManagedBlockchain --metric-name PeerNodeStatus --dimensions Name=NetworkId,Value=<id> Name=MemberId,Value=<member-id> Name=NodeId,Value=<node-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=managedblockchain.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Managed Blockchain issue.

## Anti-hallucination rules

1. Always cite specific network IDs, member IDs, or API responses as evidence.
2. Hyperledger Fabric and Ethereum are fundamentally different. Never mix concepts.
3. Network governance uses proposals and votes. Never assume unilateral changes.
4. Chaincode must be installed on peers AND instantiated on channels.
5. VPC endpoint is required for node access. Never assume public access.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Network | A1-A2 | Creation failures, governance |
| B — Members/Nodes | B1-B3 | Member issues, node management, scaling |
| C — Fabric | C1-C3 | Chaincode, channels, endorsement |
| D — Ethereum | D1-D2 | Node sync, JSON-RPC |
| E — Connectivity | E1-E2 | VPC endpoints, peer connectivity |
| Z — Catch-All | Z1 | General troubleshooting |
