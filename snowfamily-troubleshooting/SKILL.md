---
name: snowfamily-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Snow Family
  problems by analyzing device ordering, unlocking, data transfer,
  NFS shares, edge compute, EC2 on Snow, S3 adapter, clustering,
  networking, OTA updates, return shipping, and following structured
  runbooks. Covers Snowball, Snowball Edge, and Snowcone devices.
  Activate when: device ordering issues, unlock failures, data
  transfer errors, NFS problems, compute failures, S3 adapter errors,
  clustering issues, network problems, OTA update failures, shipping
  issues, or the user says something is wrong with Snow Family.
compatibility: >
  Requires AWS CLI or SDK access with snowball, s3, ec2, iam,
  cloudtrail, and cloudwatch permissions. Snowball Edge client
  required for device-level operations.
---

# AWS Snow Family Diagnostics

## When to use

Any AWS Snow Family investigation — device ordering, unlocking, data transfer, NFS shares, Snowball Edge compute, EC2 on Snow, S3 adapter, clustering, network configuration, optics, OTA updates, return shipping, or Snowcone-specific issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws snowball list-jobs --query 'JobListEntries[*].{JobId:JobId,State:JobState,Type:JobType,CreationDate:CreationDate}'
aws snowball describe-job --job-id <job-id>
aws snowball list-cluster-jobs --cluster-id <cluster-id>
```

### Step 2 — Domain deep dive

```
aws snowball get-job-manifest --job-id <job-id>
aws snowball get-job-unlock-code --job-id <job-id>
snowballEdge describe-device --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>
```

### Step 3 — Detailed investigation

```
snowballEdge describe-service --service-id s3 --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>
snowballEdge list-ec2-instances --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=snowball.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Snow Family issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `snowball list-jobs` | List all Snow Family jobs |
| `snowball describe-job` | Get job details and status |
| `snowball get-job-manifest` | Download device manifest |
| `snowball get-job-unlock-code` | Retrieve unlock code |
| `snowballEdge describe-device` | Get device status (on-device) |
| `snowballEdge describe-service` | Check service status on device |
| `snowballEdge list-ec2-instances` | List EC2 instances on device |

## Gotchas: AWS Snow Family

- Snowball Edge client commands run LOCALLY on the device, not through AWS API. You must have network access to the device IP.
- Manifest files and unlock codes are job-specific. They expire when the job is completed or cancelled.
- S3 adapter on Snowball Edge is NOT the same as Amazon S3. It supports a subset of S3 API operations.
- EC2 instances on Snowball Edge use device-specific AMIs (sbe1 instance types). Standard EC2 AMIs do not work.
- Snowball Edge clusters require a minimum of 5 devices. Individual devices cannot join or leave a cluster after creation.
- OTA updates require the device to have internet connectivity. Disconnected devices cannot receive updates.
- Snowcone has significantly less storage (14 TB HDD or 14 TB SSD) compared to Snowball Edge (80 TB or 210 TB).

## Anti-hallucination rules

1. Always cite specific job IDs, device status, or Snowball Edge client output as evidence.
2. Snowball Edge client commands and AWS CLI commands are different tools. Never conflate them.
3. S3 adapter on Snow devices has limited API support. Never assume full S3 API compatibility.
4. EC2 on Snow uses sbe1 instance types. Never suggest standard EC2 instance types.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Device Setup | A1-A2 | Device ordering issues, device unlock failures |
| B — Data Transfer | B1-B2 | Data transfer errors, NFS share issues |
| C — Edge Compute | C1-C2 | Snowball Edge compute, EC2 on Snow |
| D — Storage | D1-D2 | S3 adapter errors, clustering issues |
| E — Networking | E1-E2 | Network configuration, optics connectivity |
| F — Maintenance | F1-F2 | OTA updates, device return shipping |
| G — Snowcone | G1 | Snowcone-specific issues |
| Z — Catch-All | Z1 | General troubleshooting |
