---
title: "F1 — Gateway Connectivity Issues"
description: "Diagnose network connectivity problems between gateway and AWS"
status: active
severity: HIGH
triggers:
  - "connectivity failed"
  - "cannot reach AWS"
  - "network timeout"
  - "connection refused"
owner: devops-agent
objective: "Identify and resolve network connectivity issues between gateway and AWS services"
context: "Storage Gateway requires outbound connectivity to multiple AWS endpoints on port 443. The gateway also needs DNS resolution and NTP access. VPC endpoints can be used for private connectivity. Proxy servers may interfere with gateway communication."
---

## Phase 1 — Triage

MUST:
- Verify outbound connectivity on port 443 from gateway VM
- Check DNS resolution for Storage Gateway service endpoints
- Verify gateway VM has a default route to the internet or VPC endpoint
- Check security groups and NACLs for outbound rules

SHOULD:
- Test connectivity to specific AWS endpoints: `storagegateway.<region>.amazonaws.com`
- Check if a proxy server is configured and working correctly
- Verify VPC endpoint health if using private connectivity

MAY:
- Run traceroute to identify network path issues
- Check for MTU issues (especially with VPN or Direct Connect)

## Phase 2 — Remediate

MUST:
- Ensure security groups allow outbound TCP 443
- Verify route table has appropriate routes (internet gateway or VPC endpoint)
- Fix DNS resolution if failing

SHOULD:
- Use VPC endpoints for private connectivity: `com.amazonaws.<region>.storagegateway`
- Configure proxy settings on gateway if required by network policy

MAY:
- Set up redundant network paths for high availability
- Configure MTU appropriately for the network path

## Common Issues

- symptoms: "Gateway cannot reach AWS endpoints"
  diagnosis: "Security group or NACL blocking outbound port 443."
  resolution: "Add outbound rule allowing TCP 443 to 0.0.0.0/0 or AWS IP ranges."

- symptoms: "DNS resolution failing on gateway"
  diagnosis: "Gateway DNS configuration incorrect or DNS server unreachable."
  resolution: "Configure valid DNS servers on gateway VM. Use VPC DNS if in AWS."

- symptoms: "Intermittent connectivity"
  diagnosis: "Network instability, MTU issues, or proxy timeouts."
  resolution: "Check network path stability. Adjust MTU. Review proxy configuration."

## Output Format

```yaml
root_cause: "connectivity — <specific_cause>"
evidence:
  - type: port_test
    content: "<port 443 connectivity result>"
  - type: dns_test
    content: "<DNS resolution result>"
  - type: route_config
    content: "<route table and gateway configuration>"
severity: HIGH
mitigation:
  immediate: "Restore network connectivity to AWS"
  long_term: "Implement redundant connectivity and monitoring"
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
