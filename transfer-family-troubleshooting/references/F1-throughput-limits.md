---
title: "F1 — Transfer Throughput Limits"
description: "Diagnose slow transfer speeds and throughput bottlenecks"
status: active
severity: MEDIUM
triggers:
  - "slow transfer"
  - "throughput"
  - "bandwidth"
  - "speed limit"
  - "transfer rate"
  - "throttling"
owner: devops-agent
objective: "Identify throughput bottlenecks and optimize transfer speeds"
context: "Transfer Family throughput depends on the endpoint type, backend storage performance, network path, and client configuration. S3 supports up to 5 Gbps per prefix. EFS throughput depends on the throughput mode (bursting, provisioned, elastic). VPC endpoint throughput depends on the number of subnets and ENIs. There is no built-in per-user bandwidth throttling."
---

## Phase 1 — Triage

MUST:
- Check transfer metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Transfer --metric-name BytesIn --dimensions Name=ServerId,Value=<server-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check endpoint type (affects max throughput): `aws transfer describe-server --server-id <server-id> --query 'Server.EndpointType'`
- For EFS backend, check throughput mode: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[0].{ThroughputMode:ThroughputMode,ProvisionedThroughputInMibps:ProvisionedThroughputInMibps}'`
- For EFS, check burst credit balance: `aws cloudwatch get-metric-statistics --namespace AWS/EFS --metric-name BurstCreditBalance --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check S3 request metrics for throttling: `aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name 5xxErrors --dimensions Name=BucketName,Value=<bucket> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check network bandwidth between client and server (client-side bottleneck)
- Verify the client is using optimal transfer settings (buffer sizes, concurrent connections)
- Check if S3 prefix partitioning is causing request rate limits (3,500 PUT/s per prefix)
- For VPC endpoints, check if multiple subnets/ENIs are configured for higher aggregate throughput

MAY:
- Compare transfer speeds from different client locations
- Check if encryption overhead (SSE-KMS) is impacting throughput
- Monitor S3 request latency metrics

## Phase 2 — Remediate

MUST:
- For EFS burst credit exhaustion: switch to provisioned or elastic throughput mode
- For S3 prefix throttling: distribute objects across multiple prefixes
- Ensure the client network has sufficient bandwidth for the expected transfer rate

SHOULD:
- Use multiple subnets for VPC endpoint type to increase aggregate throughput
- Optimize client configuration: increase buffer sizes, use concurrent transfers
- Use S3 Transfer Acceleration if clients are geographically distant from the S3 region

MAY:
- Implement S3 multipart upload for large files (Transfer Family handles this automatically for files > 5 GB)
- Consider using AWS DataSync for bulk data transfers instead of Transfer Family

## Common Issues

- symptoms: "Transfer speed drops to near zero during peak hours"
  diagnosis: "EFS burst credits exhausted — throughput drops to baseline."
  resolution: "Switch to provisioned or elastic throughput mode on the EFS file system."

- symptoms: "S3 uploads slow with 503 SlowDown errors in logs"
  diagnosis: "S3 request rate limit exceeded on a single prefix (3,500 PUT/s)."
  resolution: "Distribute uploads across multiple S3 prefixes to parallelize partitions."

- symptoms: "Transfer speed is consistently low regardless of file size"
  diagnosis: "Client network bandwidth is the bottleneck, not Transfer Family."
  resolution: "Test from a higher-bandwidth client. Check for network congestion or throttling."

## Output Format

```yaml
root_cause: "throughput_limit — <specific_cause>"
evidence:
  - type: transfer_metrics
    content: "<BytesIn/BytesOut metrics>"
  - type: backend_performance
    content: "<S3 or EFS throughput metrics>"
  - type: network
    content: "<network bandwidth assessment>"
severity: MEDIUM
mitigation:
  immediate: "Address the identified bottleneck"
  long_term: "Implement throughput monitoring and capacity planning"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"
