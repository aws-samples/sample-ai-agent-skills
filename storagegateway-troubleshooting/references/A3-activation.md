---
title: "A3 — Gateway Activation Issues"
description: "Diagnose Storage Gateway activation failures"
status: active
severity: HIGH
triggers:
  - "activation failed"
  - "activation key error"
  - "cannot activate gateway"
  - "activation timeout"
owner: devops-agent
objective: "Identify why gateway activation is failing and complete activation"
context: "Gateway activation connects the gateway VM to your AWS account. Activation requires port 80 (HTTP) access to the gateway from your browser and port 443 outbound from the gateway to AWS. The activation key is retrieved via HTTP and used to register the gateway. Time synchronization is critical."
---

## Phase 1 — Triage

MUST:
- Verify gateway VM is running and has a valid IP address
- Test HTTP access to gateway on port 80: `curl http://<gateway-ip>/?activationRegion=<region>&redirect_type=JSON`
- Verify outbound connectivity from gateway on port 443
- Check NTP synchronization on gateway VM (time skew > 5 min causes failures)

SHOULD:
- Verify DNS resolution from gateway to AWS endpoints
- Check if a proxy is intercepting HTTP/HTTPS traffic
- Verify the activation region matches your intended AWS region

MAY:
- Check gateway local console for activation status
- Verify no other gateway is using the same activation key

## Phase 2 — Remediate

MUST:
- Open port 80 inbound to gateway (temporarily for activation)
- Ensure port 443 outbound is open from gateway
- Sync NTP on gateway VM
- Activate: `aws storagegateway activate-gateway --activation-key <key> --gateway-name <name> --gateway-timezone <tz> --gateway-type <FILE_S3|CACHED|STORED|VTL> --gateway-region <region>`

SHOULD:
- Close port 80 after successful activation
- Configure cache and upload buffer disks after activation: `aws storagegateway add-cache --gateway-arn <arn> --disk-ids <disk-id>`

MAY:
- Use VPC endpoint for activation in private subnets
- Automate activation with scripts

## Common Issues

- symptoms: "Cannot retrieve activation key from gateway"
  diagnosis: "Port 80 blocked or gateway VM not fully booted."
  resolution: "Open port 80 inbound. Wait for gateway VM to complete boot (5-10 minutes)."

- symptoms: "Activation key expired"
  diagnosis: "Activation key is valid for only 30 minutes after generation."
  resolution: "Generate a new activation key and activate immediately."

- symptoms: "Activation fails with time sync error"
  diagnosis: "Gateway VM clock is more than 5 minutes off from AWS."
  resolution: "Configure NTP on gateway VM. Sync time before retrying activation."

## Output Format

```yaml
root_cause: "activation_failure — <specific_cause>"
evidence:
  - type: port_access
    content: "<port 80 and 443 connectivity>"
  - type: time_sync
    content: "<NTP synchronization status>"
  - type: activation_attempt
    content: "<activation key retrieval result>"
severity: HIGH
mitigation:
  immediate: "Fix connectivity/time sync and retry activation"
  long_term: "Document activation prerequisites and automate"
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
