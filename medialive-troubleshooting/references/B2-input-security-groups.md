# B2 — MediaLive Input Security Group Issues

## Symptoms
- PUSH input receives no data despite source sending
- Input endpoint unreachable from encoder
- Silent packet drops

## Triage Commands

```bash
aws medialive list-input-security-groups
aws medialive describe-input-security-group --input-security-group-id <id>

# Check which inputs use this security group
aws medialive describe-input --input-id <id> | jq '.SecurityGroups'

# Verify source IP
curl -s https://checkip.amazonaws.com
```

## Common Causes

1. **Source IP not whitelisted** — CIDR range doesn't include encoder's public IP
2. **CIDR too restrictive** — /32 blocks single IP; dynamic IPs need wider range
3. **Security group not attached** — Input created without security group reference
4. **Multiple sources, single CIDR** — Redundant encoders need all IPs whitelisted

## Resolution

- Add source IP CIDR: `aws medialive update-input-security-group --input-security-group-id <id> --whitelist-rules '[{"Cidr":"<ip>/32"}]'`
- For dynamic IPs, use a wider CIDR range (e.g., /24)
- Verify security group is attached to the input
- For STANDARD channels, whitelist both primary and backup encoder IPs
- Note: 0.0.0.0/0 allows all IPs — use only for testing, never production


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** stop a live channel during an active broadcast without explicit customer authorization
- **NEVER** modify input security groups to remove IP allowlisting on production channels
- **NEVER** delete input attachments on running channels without a verified failover path
- **NEVER** change encoding profiles on a live channel without scheduling a maintenance window
- **NEVER** disable pipeline redundancy on channels serving production live events

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
