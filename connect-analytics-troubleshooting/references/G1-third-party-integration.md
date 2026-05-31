---
title: "G1 — Third-Party Integration"
description: "Diagnose third-party integration issues with Contact Lens and Wisdom"
status: active
severity: MEDIUM
triggers: ["integration error", "third-party connector failed", "CRM integration", "external system error"]
owner: devops-agent
objective: "Resolve third-party integration issues with Contact Lens analytics and Wisdom knowledge sources"
context: "Contact Lens data can be exported to third-party analytics tools. Wisdom supports connectors for ServiceNow, Salesforce, SharePoint, and Zendesk. Integrations require proper authentication and network connectivity."
---

## Phase 1 — Triage

MUST:
- Identify the third-party system and integration type (Wisdom connector, data export, API integration)
- Check connector status for Wisdom knowledge sources: `aws wisdom list-knowledge-bases`
- Verify IAM permissions for the integration service role

SHOULD:
- Check connector credentials and authentication status
- Verify network connectivity between AWS and the third-party system
- Review CloudTrail for integration-related API errors

MAY:
- Check the third-party system's logs for connection errors
- Test API connectivity to the third-party endpoint
- Review integration documentation for version compatibility

## Phase 2 — Remediate

MUST:
- Fix authentication credentials for third-party connectors
- Verify network path between AWS and the third-party system (VPC endpoints, NAT, firewall)
- Ensure the third-party system's API is accessible and responding

SHOULD:
- Update connector credentials if they have expired or been rotated
- Verify the third-party API version is compatible with the AWS connector
- Test the integration with a small data set before full sync

MAY:
- Configure retry logic for transient integration failures
- Set up monitoring for integration health
- Implement fallback procedures for integration outages

## Safety Ratings

```
safety_ratings:
  - "list-knowledge-bases: GREEN — read-only enumeration"
  - "connector credential update: RED — modifies authentication to external system"
  - "network configuration changes: RED — affects connectivity to external systems"
  - "third-party API testing: YELLOW — sends requests to external system"
```

## Escalation Conditions

- Third-party system is down and no fallback is available
- Connector credentials expired and renewal requires third-party admin access
- Integration data format changed due to third-party API update
- Data sync between AWS and third-party system is inconsistent

## Data Sensitivity

- `aws wisdom list-knowledge-bases`: LOW — returns knowledge base metadata
- Third-party connector credentials: CRITICAL — authentication secrets
- Data exchanged with third-party systems: HIGH — may contain customer interaction data
- Third-party API endpoints: MEDIUM — reveals integration architecture

## Prohibited Actions

- NEVER store third-party credentials in plain text or logs
- NEVER send customer data to unauthorized third-party systems
- NEVER bypass authentication for third-party integrations
- NEVER modify third-party system configuration without the system owner's approval

## Phase 3 — Rollback

1. Restore previous connector credentials if rotation caused the issue
2. Revert network changes if they broke connectivity to the third-party system
3. Disable the integration temporarily if it is causing data issues
4. Verify data consistency between AWS and the third-party system after changes
5. Document the integration issue and resolution for both teams

## Common Issues

- symptoms: "Wisdom ServiceNow connector shows sync failed"
  diagnosis: "ServiceNow API credentials expired or were rotated."
  resolution: "Update the connector credentials with current ServiceNow API credentials."

## Output Format

```yaml
root_cause: "third_party_integration — <specific_cause>"
evidence:
  - type: integration_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix integration connectivity or authentication"
  long_term: "Implement integration monitoring and credential rotation procedures"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
