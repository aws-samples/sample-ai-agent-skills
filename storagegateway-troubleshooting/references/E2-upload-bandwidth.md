---
title: "E2 — Upload Bandwidth Issues"
description: "Diagnose upload bandwidth bottlenecks and rate limiting"
status: active
severity: MEDIUM
triggers:
  - "upload slow"
  - "bandwidth limit"
  - "upload buffer full"
  - "data not reaching S3"
owner: devops-agent
objective: "Identify and resolve upload bandwidth issues affecting data durability"
context: "Gateway uploads data to S3 asynchronously. Upload bandwidth affects how quickly local data is durably stored in S3. When upload buffer fills, the gateway stops accepting writes. Bandwidth rate limits can be configured for upload and download independently. Network congestion and WAN limitations are common bottlenecks."
---

## Phase 1 — Triage

MUST:
- Check bandwidth limits: `aws storagegateway describe-bandwidth-rate-limit --gateway-arn <gateway-arn>`
- Check upload buffer: `aws cloudwatch get-metric-statistics --namespace AWS/StorageGateway --metric-name UploadBufferPercentUsed --dimensions Name=GatewayId,Value=<gw-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check CloudBytesUploaded metric for actual upload throughput
- Verify network bandwidth between gateway and AWS

SHOULD:
- Check if bandwidth rate limit is set too low
- Monitor CachePercentDirty (indicates pending uploads)
- Test network throughput to AWS endpoints

MAY:
- Check for network congestion on the WAN link
- Review if other applications are competing for bandwidth

## Phase 2 — Remediate

MUST:
- Remove or increase bandwidth limit if too restrictive: `aws storagegateway update-bandwidth-rate-limit --gateway-arn <arn> --average-upload-rate-limit-in-bits-per-sec 104857600`
- Add upload buffer capacity if consistently full
- Ensure network path to AWS has sufficient bandwidth

SHOULD:
- Schedule bandwidth limits for off-peak hours: `aws storagegateway update-bandwidth-rate-limit-schedule --gateway-arn <arn> --bandwidth-rate-limit-intervals <intervals>`
- Monitor UploadBufferPercentUsed with CloudWatch alarms

MAY:
- Consider AWS Direct Connect for dedicated bandwidth
- Implement QoS policies to prioritize gateway traffic

## Common Issues

- symptoms: "Upload buffer at 100%, writes failing"
  diagnosis: "Upload bandwidth insufficient to keep up with write rate."
  resolution: "Increase bandwidth limit. Add upload buffer capacity. Reduce write rate."

- symptoms: "Data not appearing in S3 promptly"
  diagnosis: "Upload backlog due to bandwidth constraints."
  resolution: "Check bandwidth limits. Monitor CloudBytesUploaded. Increase bandwidth."

- symptoms: "Bandwidth limit set but actual throughput much lower"
  diagnosis: "Network bottleneck below the configured limit."
  resolution: "Test network throughput. Check for congestion. Consider Direct Connect."

## Output Format

```yaml
root_cause: "upload_bandwidth — <specific_cause>"
evidence:
  - type: bandwidth_limits
    content: "<configured upload/download limits>"
  - type: buffer_utilization
    content: "<UploadBufferPercentUsed>"
  - type: actual_throughput
    content: "<CloudBytesUploaded metric>"
severity: MEDIUM
mitigation:
  immediate: "Adjust bandwidth limits and add buffer capacity"
  long_term: "Right-size network bandwidth for gateway workload"
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

- Gateway volumes contain production application data
- Tape archives may contain regulated backup data
- File shares expose S3 bucket contents to on-premises systems
- Classification: CONFIDENTIAL - production data in transit and at rest

## Prohibited Actions

- NEVER shut down or reboot gateway VMs during active data transfers
- NEVER delete volumes or tapes containing unverified backup data
- NEVER modify cache disk configuration on production gateways without maintenance window
- NEVER change iSCSI targets or NFS/SMB shares during business hours without approval

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
