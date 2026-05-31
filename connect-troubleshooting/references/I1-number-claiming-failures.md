---
title: "I1 — Phone Number Claiming Failures"
description: "Diagnose failures when claiming phone numbers for a Connect instance"
status: active
severity: HIGH
triggers:
  - "Cannot claim number"
  - "Number not available"
  - "Phone number error"
  - "DID/toll-free claiming failed"
owner: devops-agent
objective: "Successfully claim phone numbers for the Connect instance"
context: "Phone numbers are claimed from the available inventory in the instance's region. Availability varies by country, type (DID/toll-free), and region. Some countries require documentation. Numbers are region-bound and cannot be moved between regions."
---

## Phase 1 — Triage

MUST:
- Check available number inventory for the desired country and type
- Verify the instance is in the correct region for the desired numbers
- Check if the country requires regulatory documentation
- Verify service quotas for phone numbers haven't been reached

SHOULD:
- Check if the specific number type (DID, toll-free) is available in the region
- Verify the instance has the required telephony features enabled
- Check if there are pending number orders

MAY:
- Check AWS service quotas for Connect phone numbers
- Review documentation requirements for the target country

## Phase 2 — Remediate

MUST:
- If no inventory: try a different area code or number type
- If documentation required: submit the required regulatory documents
- If quota reached: request a quota increase

SHOULD:
- Plan number procurement in advance for new regions/countries
- Document the number claiming process and requirements

MAY:
- Request specific number ranges through AWS Support
- Implement number inventory monitoring

## Common Issues

- symptoms: "No numbers available in desired area code"
  diagnosis: "Number inventory exhausted for that area code."
  resolution: "Try a different area code or request specific numbers through AWS Support."

- symptoms: "Country requires documentation"
  diagnosis: "Regulatory requirements for the target country."
  resolution: "Submit required documentation (business registration, address proof, etc.)."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
```

## Escalation Conditions

- Contact center is live with active calls or chats in progress
- Issue affects agent ability to receive or handle contacts
- Fix requires modifying a contact flow associated with a phone number
- Resolution involves changes to hours of operation during business hours
- Issue impacts call recording, CTR generation, or compliance requirements

## Data Sensitivity

HIGH - Contact Trace Records (CTRs) contain customer phone numbers, call timestamps, and interaction details. Call recordings contain customer voice data. Chat transcripts may contain PII, financial data, or healthcare information. Agent data includes employee performance metrics.

## Prohibited Actions

- NEVER suggest deleting a contact flow that is associated with a phone number - this breaks inbound routing
- NEVER suggest modifying production hours of operation during active business hours
- NEVER recommend removing security profiles from agents while they are logged in
- NEVER suggest disabling call recording on queues that have compliance requirements
- NEVER recommend deleting or modifying a published contact flow without first testing in a non-production flow

## Phase 3 - Rollback

1. If agent routing or security profiles were modified, restore original profile assignments
2. If phone number associations were changed, reassociate the number with the original contact flow
## Output Format

```yaml
root_cause: "number_claiming — <specific cause>"
evidence:
  - type: number_inventory
    content: "<availability details>"
severity: HIGH
mitigation:
  immediate: "Try alternative numbers or submit documentation"
  long_term: "Plan number procurement and monitor inventory"
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
