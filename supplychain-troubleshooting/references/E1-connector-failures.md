---
title: "E1 — Connector Failures"
description: "Diagnose Supply Chain connector configuration and execution failures"
status: active
severity: HIGH
triggers: ["connector failed", "SAP connector error", "EDI connector", "connector configuration"]
owner: devops-agent
objective: "Resolve connector failures in Supply Chain"
context: "Connectors integrate external systems (SAP, EDI) with Supply Chain. Each connector type has specific prerequisites."
---

## Phase 1 — Triage

MUST:
- Verify instance is ACTIVE: `aws scn get-instance --instance-id <instance-id>`
- Check data integration flow status for connector-based flows: `aws scn list-data-integration-flows --instance-id <instance-id>`
- Verify connector credentials and network connectivity

SHOULD:
- Check IAM role permissions for the connector
- Verify source system is accessible from the AWS network

MAY:
- Review CloudTrail for connector-related API calls
- Check VPC configuration if using private connectivity

## Phase 2 — Remediate

MUST:
- Fix connector credentials (rotate if expired)
- Fix network connectivity (security groups, NACLs, VPC peering)
- Update IAM role permissions for connector access

SHOULD:
- Implement credential rotation schedules
- Set up connector health monitoring

MAY:
- Use AWS PrivateLink for secure connectivity
- Implement connector failover patterns

## Safety Ratings

```
safety_ratings:
  - "get-instance: GREEN — read-only instance check"
  - "list-data-integration-flows: GREEN — read-only flow enumeration"
  - "Credential rotation: YELLOW — changes authentication, may cause temporary disruption"
  - "Network configuration changes: YELLOW — affects connectivity"
```

## Escalation Conditions

- Connector failures affect production data integration
- Credential issues require source system administrator involvement
- Network connectivity requires VPC or firewall changes
- Connector type not supported for the source system

## Data Sensitivity

- Connector credentials: CRITICAL — authentication secrets
- Source system data: HIGH — proprietary business data
- Network configuration: MEDIUM — infrastructure details
- Connector logs: MEDIUM — may contain data references

## Prohibited Actions

- NEVER store connector credentials in plain text
- NEVER share connector credentials in tickets or logs
- NEVER modify network security groups without change management
- NEVER bypass authentication for troubleshooting

## Phase 3 — Rollback

1. If connector credentials were rotated: verify old credentials are revoked
2. If network configuration was changed: revert security group and NACL changes
3. If connector was reconfigured: restore previous configuration
4. Verify connector is functioning after changes
5. Document all changes for security audit

## Common Issues

- symptoms: "Connector returns authentication error"
  diagnosis: "Connector credentials are expired or incorrect."
  resolution: "Rotate credentials in the source system and update connector configuration."

- symptoms: "Connector timeout connecting to source system"
  diagnosis: "Network connectivity issue between AWS and source system."
  resolution: "Check security groups, NACLs, and routing. Verify VPC peering or VPN connectivity."

## Output Format

```yaml
root_cause: "connector — <specific_cause>"
evidence:
  - type: connector_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix connector issue"
  long_term: "Implement connector monitoring and credential rotation"
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
