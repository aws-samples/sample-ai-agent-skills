---
name: network-manager-troubleshooting
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Network Manager
  and Cloud WAN problems by analyzing global networks, core network
  policies, attachments, peering, routing, segments, transit gateway
  registration, VPN attachments, telemetry, and policy versioning
  using structured runbooks.
  Activate when: global network creation failures, core network policy
  errors, attachment failures, peering issues, route analysis errors,
  segment configuration problems, transit gateway registration issues,
  VPN attachment failures, telemetry gaps, or the user says something
  is wrong with Network Manager or Cloud WAN.
compatibility: >
  Requires AWS CLI or SDK access with networkmanager, ec2,
  cloudwatch, cloudtrail, and organizations permissions.
---

# AWS Network Manager / Cloud WAN Diagnostics

## When to use

Any AWS Network Manager or Cloud WAN investigation — global network creation, core network policy errors, attachment failures, peering issues, route analysis, routing policies, segment configuration, network function groups, transit gateway registration, site-to-site VPN attachment, telemetry, CloudWatch integration, policy version management, or general troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws networkmanager describe-global-networks --query 'GlobalNetworks[*].{Id:GlobalNetworkId,State:State,Description:Description}'
aws networkmanager list-core-networks --query 'CoreNetworks[*].{Id:CoreNetworkId,State:State,Description:Description}'
aws networkmanager list-attachments --query 'Attachments[*].{Id:AttachmentId,Type:AttachmentType,State:State,EdgeLocation:EdgeLocation}'
```

### Step 2 — Domain deep dive

```
aws networkmanager get-core-network --core-network-id <id> --query 'CoreNetwork.{State:State,Edges:Edges,Segments:Segments}'
aws networkmanager get-core-network-policy --core-network-id <id>
aws networkmanager get-network-routes --global-network-id <gn-id> --route-table-identifier Type=CORE_NETWORK_SEGMENT,CoreNetworkId=<cn-id>,SegmentName=<segment>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=networkmanager.amazonaws.com --max-results 20
aws networkmanager get-core-network-change-set --core-network-id <id> --policy-version-id <version>
aws networkmanager list-core-network-policy-versions --core-network-id <id>
```

Read `references/guardrails.md` before concluding on any Network Manager or Cloud WAN issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `networkmanager describe-global-networks` | List global networks |
| `networkmanager get-core-network` | Get core network details and edges |
| `networkmanager get-core-network-policy` | Retrieve active or pending policy |
| `networkmanager list-attachments` | List all attachments |
| `networkmanager get-network-routes` | Retrieve routes for a segment |
| `networkmanager list-peerings` | List peering connections |
| `networkmanager get-transit-gateway-registrations` | Check TGW registrations |

## Gotchas: AWS Network Manager / Cloud WAN

- Core network policies are JSON documents with segments, attachment-policies, and segment-actions. A policy must be EXECUTED (not just created) to take effect.
- Policy changes generate a change set. You must review and execute the change set — it is NOT auto-applied.
- Attachments go through a lifecycle: CREATING → AVAILABLE → UPDATING → DELETING. Failures can leave attachments in FAILED or REJECTED state.
- Peering between core networks requires both sides to accept. A peering stuck in CREATING may indicate the remote side has not accepted.
- Route analysis operates on the EXECUTED policy. If you changed the policy but did not execute, routes reflect the old policy.
- Transit gateway registration is per-region. A TGW must be registered in the global network before it can be used as an attachment.
- Telemetry data (bandwidth, packet loss, latency) requires CloudWatch agent or VPN tunnel metrics. Missing telemetry usually means the data source is not configured.

## Anti-hallucination rules

1. Always cite specific global network IDs, core network IDs, attachment IDs, or API responses as evidence.
2. Core network policy changes require explicit execution. Never claim a policy is active just because it was created.
3. Attachment states have a defined lifecycle. Never skip states or assume instant transitions.
4. Route analysis reflects the EXECUTED policy, not pending changes. Never conflate pending and active policies.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Global Network | A1–A2 | Global network creation, core network policy errors |
| B — Attachments | B1–B2 | Attachment failures, peering issues |
| C — Routing | C1–C2 | Route analysis errors, routing policy issues |
| D — Segments | D1–D2 | Segment configuration, network function groups |
| E — Transit Gateway | E1–E2 | TGW registration, site-to-site VPN attachment |
| F — Telemetry | F1–F2 | Telemetry issues, CloudWatch integration |
| G — Policy Management | G1 | Policy version management |
| Z — Catch-All | Z1 | General troubleshooting |
