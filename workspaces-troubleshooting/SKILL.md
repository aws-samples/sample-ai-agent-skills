---
name: workspaces-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon WorkSpaces problems
  by analyzing provisioning, connectivity, performance, directory integration,
  networking, management operations, and security configurations following
  structured runbooks. Activate when: WorkSpace launch failures, bundle
  issues, directory integration problems, client connection failures,
  streaming protocol issues, MFA failures, latency or bandwidth problems,
  AD Connector or AWS Managed AD issues, VPC configuration errors, internet
  access problems, rebuild/restore failures, maintenance window issues,
  encryption problems, access control issues, or the user says something
  is wrong with WorkSpaces without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with WorkSpaces, Directory Service, IAM,
  EC2, CloudWatch, and optionally CloudTrail permissions.
---

# WorkSpaces Diagnostics

## When to use

Any WorkSpaces investigation where the console alone is insufficient — launch failures, client connectivity issues, streaming protocol problems, MFA configuration, performance degradation, directory integration errors, VPC networking, rebuild/restore operations, or security configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws workspaces describe-workspaces --workspace-id <id>
aws workspaces describe-workspace-directories
aws workspaces describe-workspace-bundles --bundle-ids <id>
aws ds describe-directories
aws cloudwatch get-metric-statistics --namespace AWS/WorkSpaces --metric-name Available --dimensions Name=WorkspaceId,Value=<id> --start-time <iso> --end-time <iso> --period 300 --statistics Average
```

### Step 2 — Domain deep dive

```
aws workspaces describe-workspace-images
aws workspaces describe-connection-status --workspace-id <id>
aws workspaces describe-workspace-snapshots --workspace-id <id>
aws ds describe-conditional-forwarders --directory-id <id>
aws ec2 describe-security-groups --group-ids <sg-id>
aws ec2 describe-subnets --subnet-ids <subnet-id>
```

Read `references/guardrails.md` before concluding on any WorkSpaces issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `workspaces describe-workspaces` | WorkSpace state, bundle, directory, IP |
| `workspaces describe-workspace-directories` | Directory registration, subnets, security |
| `workspaces describe-workspace-bundles` | Bundle config, compute type, storage |
| `workspaces describe-connection-status` | Last known client connection state |
| `workspaces describe-workspace-images` | Custom image status and errors |
| `ds describe-directories` | Directory type, status, DNS, VPC |
| `ds describe-conditional-forwarders` | DNS forwarding to on-premises AD |
| `ec2 describe-security-groups` | Security group rules for WorkSpaces |
| `ec2 describe-subnets` | Subnet CIDR, AZ, available IPs |

## Gotchas: WorkSpaces

- WorkSpaces require at least two subnets in different AZs for directory registration, even if you only launch in one AZ.
- The WorkSpaces client uses port 4172 (TCP and UDP) for PCoIP and port 4195 (TCP and UDP) for WSP. Both must be open in security groups and NACLs.
- WorkSpaces have a 20-minute health check interval. A WorkSpace showing UNHEALTHY may recover on the next check cycle.
- Custom bundles created from a WorkSpace inherit the source WorkSpace's volume sizes. You cannot reduce volume size after creation.
- MFA with AD Connector requires a RADIUS server. AWS Managed Microsoft AD supports smart card and RADIUS-based MFA.
- WorkSpaces in ADMIN_MAINTENANCE state cannot be accessed by users. This state is used for patching and updates.
- Rebuilding a WorkSpace restores the root volume from the latest bundle image and the user volume from the last snapshot.
- WorkSpaces auto-stop mode stops the WorkSpace after a configurable idle timeout (default 1 hour). Users must reconnect to restart.
- IP access control groups are allowlists. If configured, only IPs in the group can connect. An empty group blocks all connections.
- WorkSpaces Web Access requires WSP protocol. PCoIP does not support browser-based access.

## Anti-hallucination rules

1. Always cite specific WorkSpace state, connection status, directory configuration, or CloudWatch metrics as evidence.
2. UNHEALTHY state does not mean the WorkSpace is permanently broken. Always check if it recovers on the next health check.
3. Never claim PCoIP supports Web Access. Only WSP protocol supports browser-based connections.
4. WorkSpaces require two subnets in different AZs. Never suggest single-subnet directory registration.
5. Rebuilding preserves user volume data from the last snapshot, not real-time data. Never claim rebuild preserves all current user data.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Provisioning | A1–A3 | Launch failures, bundle issues, directory integration |
| B — Connectivity | B1–B3 | Client connection, streaming protocol, MFA |
| C — Performance | C1–C2 | Latency, bandwidth |
| D — Directory | D1–D2 | AD Connector, AWS Managed AD |
| E — Networking | E1–E2 | VPC config, internet access |
| F — Management | F1–F2 | Rebuild, restore, maintenance |
| G — Security | G1–G2 | Encryption, access control |
| Z — Catch-All | Z1 | General troubleshooting |
