---
name: ec2-instance-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot EC2 instance problems
  by collecting OS-level diagnostic data and following structured
  runbooks. Activate when: an instance is unreachable or unresponsive,
  instance status checks are failing (system or instance status),
  instance is stuck in pending/stopping/shutting-down state, SSH or
  RDP connections fail, instance performance is degraded (high CPU,
  memory, disk, or network), EBS volumes won't attach or mount,
  instance store data is lost, networking issues (ENI, security group,
  NACL, VPC routing), IAM/IMDS credential failures, instance launch
  failures (InsufficientInstanceCapacity, limit exceeded), boot or
  initialization failures (user-data, cloud-init), or the user says
  something is wrong with an EC2 instance without naming specific
  symptoms. Also activate when the user wants to inspect instance-level
  artifacts like system logs, screenshot, network configuration,
  metadata, or storage configuration.
compatibility: >
  Requires AWS CLI or SDK access with appropriate EC2, CloudWatch, and
  SSM permissions. For OS-level diagnostics, instances must have SSM
  Agent running or SSH access available.
---

# EC2 Instance Diagnostics

## When to use

Any EC2 instance investigation where the AWS Console alone is insufficient — system logs, instance screenshots, OS-level metrics, network configuration, storage state, boot failures, performance issues, or connectivity problems.

## Investigation workflow

### Step 1 — Collect and triage

```
# Get instance status and details
aws ec2 describe-instances --instance-ids <id>
aws ec2 describe-instance-status --instance-ids <id>

# Get system log (serial console output)
aws ec2 get-console-output --instance-id <id>

# Get instance screenshot (for boot issues)
aws ec2 get-console-screenshot --instance-id <id>

# If SSM is available:
aws ssm send-command --instance-ids <id> --document-name "AWS-RunShellScript" --parameters commands=["dmesg","journalctl -b --no-pager -n 500"]
```

Triage returns:
- Instance state and status checks
- System log with boot messages and kernel output
- Screenshot for visual boot state
- SSM command output for live OS diagnostics

If status checks are failing, that IS the root cause domain. Don't chase application-level symptoms.

### Step 2 — Domain deep dive (only if needed)

```
# Networking
aws ec2 describe-network-interfaces --filters Name=attachment.instance-id,Values=<id>
aws ec2 describe-security-groups --group-ids <sg-ids>
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>

# Storage
aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=<id>
aws ec2 describe-volume-status --volume-ids <vol-ids>

# Performance (CloudWatch)
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization ...
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name StatusCheckFailed ...

# Live OS diagnostics (via SSM)
aws ssm send-command --document-name "AWS-RunShellScript" --parameters commands=["top -bn1","free -m","df -h","ip addr","ss -tlnp"]
```

Read `references/ec2-storage-guardrails.md` before concluding on any storage issue.
Read `references/ec2-networking-guardrails.md` before making any networking behavioral claim.

### Step 3 — Detailed path (low-confidence cases only)

```
# CloudWatch Logs (if agent installed)
aws logs filter-log-events --log-group-name <group> --filter-pattern "ERROR"

# CloudTrail for API-level events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=<id>

# EC2 serial console (interactive, for unresponsive instances)
aws ec2-instance-connect send-serial-console-ssh-public-key --instance-id <id>
```

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-instances` | Instance state, type, VPC, subnet, SGs, IAM role |
| `describe-instance-status` | System and instance status checks |
| `get-console-output` | Serial console / system log (boot issues, kernel panics) |
| `get-console-screenshot` | Visual boot state (BSOD, GRUB, fsck) |
| `describe-network-interfaces` | ENI details, IPs, SGs per interface |
| `describe-security-groups` | Inbound/outbound rules |
| `describe-network-acls` | Subnet-level allow/deny rules |
| `describe-route-tables` | VPC routing (IGW, NAT, peering, TGW) |
| `describe-volumes` | EBS volume state, attachment, type, IOPS |
| `describe-volume-status` | EBS volume health checks |
| `get-metric-statistics` | CloudWatch metrics (CPU, network, disk, status) |
| `ssm send-command` | Run OS commands via SSM Agent |
| `describe-instance-types` | Instance capabilities, limits, features |
| `describe-instance-attribute` | User-data, kernel, ramdisk, instance-initiated-shutdown |
| `monitor-instances` | Enable/check detailed monitoring |

## Gotchas: Instance Status Checks

These are the mistakes commonly made during EC2 troubleshooting.

- System status check failure means the UNDERLYING HOST has a problem. You cannot fix this from inside the instance. Stop and start (NOT reboot) to migrate to new hardware.
- Instance status check failure means the OS/software inside the instance has a problem. Check system log and screenshot.
- Reboot does NOT change the underlying host. Only stop+start migrates to new hardware.
- Stop+start CHANGES the public IP (unless using Elastic IP). Private IP is preserved.
- Instance store volumes are EPHEMERAL. Data is lost on stop/start, termination, or host failure.
- Nitro instances have different system log behavior than Xen instances. Nitro provides serial console access.
- T-series instances can become unresponsive when CPU credits are exhausted (T2) or baseline is exceeded without credits (T3 unlimited disabled).
- Enhanced networking (ENA) must be enabled in both the AMI and instance type. Mismatch causes launch failure.
- IMDSv2 hop limit of 1 blocks containers and multi-hop network paths from reaching metadata.

### Status check ownership

| Check | Owner | Fix |
|-------|-------|-----|
| System status check | AWS (underlying host) | Stop + Start (migrate hardware) |
| Instance status check | Customer (OS/software) | Fix OS, check system log |
| EBS volume status | AWS/Customer | Check volume status, detach/reattach |

### Instance state transitions

| From | To | Trigger |
|------|----|---------|
| pending | running | Launch complete |
| running | stopping | Stop requested |
| stopping | stopped | Stop complete |
| stopped | pending | Start requested |
| running | shutting-down | Terminate requested |
| shutting-down | terminated | Terminate complete |
| any | stuck | Underlying issue (see runbooks) |

## Gotchas: Networking

- Security groups are STATEFUL (return traffic auto-allowed). NACLs are STATELESS (must allow both directions).
- Security group default: deny all inbound, allow all outbound. NACL default: allow all both directions.
- You CANNOT block specific IPs with security groups (no deny rules). Use NACLs for deny rules.
- Source/destination check must be DISABLED for NAT instances, load balancers, or any traffic forwarding.
- Elastic IPs are free when attached to a running instance. They cost money when unattached or attached to a stopped instance.
- VPC flow logs show ACCEPT/REJECT but NOT the content. They log at the ENI level.
- MTU: VPC default is 1500. Jumbo frames (9001) only work within the same VPC/region and require instance support.
- DNS resolution in VPC requires enableDnsSupport=true. DNS hostnames require enableDnsHostnames=true.

## Gotchas: Storage

Full details in `references/ec2-storage-guardrails.md`.

- EBS root volume delete-on-termination is TRUE by default for AMI-created volumes. Additional volumes default to FALSE.
- gp3 volumes have baseline 3000 IOPS and 125 MB/s regardless of size. gp2 baseline is 3 IOPS/GB (min 100).
- io1/io2 IOPS are provisioned independently of size but capped at 50 IOPS/GB ratio.
- EBS-optimized throughput is instance-type dependent. Small instances can bottleneck large volumes.
- NVMe device names (/dev/nvme*) don't match the block device mapping names (/dev/sd*). Use `lsblk` or `nvme id-ctrl`.
- EBS Multi-Attach (io1/io2 only) requires cluster-aware filesystem. ext4/xfs will corrupt data.
- Snapshots are incremental but each snapshot is independently restorable.
- Volume modification (resize, type change, IOPS change) has a 6-hour cooldown between modifications.

## Anti-hallucination rules

1. Always cite specific AWS API responses, system log excerpts, or CloudWatch metrics as evidence.
2. Status check failures have specific ownership (system vs instance). Never confuse them.
3. Stop+start is NOT the same as reboot. They have fundamentally different effects.
4. Instance store is ephemeral. Never claim instance store data survives stop/start.
5. Security groups have no deny rules. Never suggest adding a deny rule to a security group.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 40 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Launch Failures | A1-A4 | InsufficientCapacity, limit exceeded, AMI issues, launch template errors |
| B — Status Check Failures | B1-B3 | System status, instance status, EBS status |
| C — Connectivity | C1-C5 | SSH/RDP failure, security group, NACL, routing, DNS |
| D — Performance | D1-D5 | CPU exhaustion, memory pressure, disk I/O, network throughput, credit depletion |
| E — Storage | E1-E4 | EBS attach/detach, volume full, IOPS throttling, instance store loss |
| F — Boot/Init | F1-F4 | Kernel panic, fsck, cloud-init, user-data, GRUB |
| G — Networking | G1-G4 | ENI issues, EIP, NAT, VPC peering/TGW |
| H — IAM/Security | H1-H3 | Instance profile, IMDS, KMS/encryption |
| I — State Transitions | I1-I3 | Stuck stopping, stuck pending, unexpected termination |
| J — Instance Types | J1-J3 | Nitro vs Xen, bare metal, Graviton |
| K — Maintenance | K1-K3 | Scheduled events, retirement, host recovery |
| Z — Catch-All | Z1 | General troubleshooting |
