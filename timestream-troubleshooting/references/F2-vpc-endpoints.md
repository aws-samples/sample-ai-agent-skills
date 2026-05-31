---
title: "F2 — VPC Endpoint Issues"
description: "Diagnose Timestream VPC endpoint configuration problems"
status: active
severity: MEDIUM
triggers:
  - "VPC endpoint Timestream"
  - "private access Timestream"
  - "endpoint configuration"
  - "cannot reach Timestream from VPC"
owner: devops-agent
objective: "Identify and resolve VPC endpoint issues for Timestream"
context: "Timestream supports VPC interface endpoints for private connectivity. Separate endpoints are needed for write (timestream-ingest) and query (timestream) operations. Endpoint discovery must work through the VPC endpoint."
---

## Phase 1 — Triage

MUST:
- Check if VPC endpoints exist for Timestream: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.timestream.ingest-cell1,com.amazonaws.<region>.timestream.query-cell1`
- Verify endpoint status is available
- Check security groups on the endpoint ENIs allow HTTPS (443)
- Verify DNS resolution for Timestream endpoints from within the VPC

SHOULD:
- Check if private DNS is enabled on the endpoints
- Verify the VPC has enableDnsSupport and enableDnsHostnames enabled
- Test connectivity from an instance in the VPC

MAY:
- Check VPC Flow Logs for blocked traffic to Timestream
- Verify endpoint policies allow the required operations

## Phase 2 — Remediate

MUST:
- Create VPC endpoints for both ingest and query if missing
- Configure security groups to allow HTTPS (443) inbound from VPC CIDR
- Enable private DNS on the endpoints

SHOULD:
- Ensure VPC DNS attributes are enabled
- Test endpoint discovery through the VPC endpoint

MAY:
- Configure endpoint policies for access control
- Monitor endpoint usage

## Common Issues

- symptoms: "Cannot reach Timestream from private subnet"
  diagnosis: "No VPC endpoint for Timestream, and no NAT gateway."
  resolution: "Create VPC interface endpoints for Timestream ingest and query."

- symptoms: "Endpoint exists but connection fails"
  diagnosis: "Security groups on endpoint ENIs blocking HTTPS traffic."
  resolution: "Allow inbound HTTPS (443) from the VPC CIDR in the endpoint security group."

- symptoms: "Endpoint discovery fails through VPC endpoint"
  diagnosis: "Private DNS not enabled or VPC DNS attributes not set."
  resolution: "Enable private DNS on the endpoint. Set enableDnsSupport and enableDnsHostnames on VPC."

## Output Format

```yaml
root_cause: "vpc_endpoint_issue — <specific_cause>"
evidence:
  - type: endpoint_status
    content: "<VPC endpoint state>"
  - type: security_groups
    content: "<security group rules>"
  - type: dns_config
    content: "<private DNS and VPC DNS settings>"
severity: MEDIUM
mitigation:
  immediate: "Fix VPC endpoint configuration"
  long_term: "Automate VPC endpoint setup with IaC"
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

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

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
