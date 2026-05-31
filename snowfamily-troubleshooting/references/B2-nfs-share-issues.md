---
title: "B2 — NFS Share Issues"
description: "Diagnose AWS Snow Family NFS file interface issues"
status: active
severity: MEDIUM
triggers: ["NFS mount failed", "NFS share error", "file transfer NFS", "NFS permission denied"]
owner: devops-agent
objective: "Resolve NFS file interface issues including mount failures, permissions, and file compatibility"
context: "NFS interface provides file-level access to Snow device storage. Supports NFSv3 and NFSv4. Has file size and name restrictions."
---

## Phase 1 — Triage

MUST:
- Check NFS service status: `snowballEdge describe-service --service-id fileinterface --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Verify NFS endpoint is accessible: `showmount -e <device-ip>`
- Check mount status on client: `mount | grep <device-ip>`

SHOULD:
- Verify client NFS packages are installed (nfs-utils or nfs-common)
- Check firewall rules allow NFS traffic (ports 111, 2049)

MAY:
- Review NFS export configuration on the device
- Test with a small file transfer to isolate the issue

## Phase 2 — Remediate

MUST:
- Start NFS service if not running: `snowballEdge start-service --service-id fileinterface --virtual-network-interface-arns <vni-arn>`
- Fix mount command syntax: `mount -t nfs <device-ip>:/buckets/<bucket-name> /mnt/snowball`
- Ensure NFS client packages are installed on the transfer host

SHOULD:
- Use NFSv3 for better compatibility: `mount -t nfs -o vers=3 <device-ip>:/buckets/<bucket-name> /mnt/snowball`
- Verify file names comply with Snow device restrictions

MAY:
- Configure NFS mount options for performance (rsize, wsize, async)

## Safety Ratings

```
safety_ratings:
  - "describe-service: GREEN — read-only service status"
  - "showmount: GREEN — read-only NFS export listing"
  - "start-service: YELLOW — starts NFS service on device"
  - "mount: YELLOW — mounts device storage on client"
```

## Escalation Conditions

- NFS service fails to start after multiple attempts
- File transfer corrupts data during NFS copy
- NFS mount hangs or becomes unresponsive
- File name restrictions prevent transfer of required files

## Data Sensitivity

- `snowballEdge describe-service`: LOW — reveals service configuration
- `showmount -e`: LOW — reveals NFS export paths
- NFS mounted data: HIGH — direct access to transferred data

## Prohibited Actions

- NEVER mount NFS shares with write access from multiple clients simultaneously
- NEVER transfer files with unsupported characters without renaming first
- NEVER unmount NFS shares during active file transfers
- NEVER bypass file size limits by splitting files without a reassembly plan

## Phase 3 — Rollback

1. Unmount NFS share cleanly: `umount /mnt/snowball`
2. If NFS service was started with wrong configuration, stop and restart it
3. Verify no partial files remain on the device from interrupted transfers
4. Restore original NFS mount configuration on the client
5. Document NFS configuration and mount options for future transfers

## Common Issues

- symptoms: "mount.nfs: access denied by server when mounting"
  diagnosis: "NFS file interface service is not running or virtual network interface is not configured."
  resolution: "Start the file interface service with a virtual network interface ARN. Verify the export path includes /buckets/<bucket-name>."

## Output Format

```yaml
root_cause: "nfs_share — <specific_cause>"
evidence:
  - type: nfs_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix NFS configuration"
  long_term: "Document NFS procedures"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
