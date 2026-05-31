# D2 — DataZone Subscription Fulfillment Issues

## Symptoms
- Approved subscription not granting access
- Data not accessible after subscription
- Subscription fulfillment errors

## Triage Commands

```bash
aws datazone get-subscription --domain-identifier <domain-id> --identifier <sub-id>
aws datazone list-subscription-grants --domain-identifier <domain-id> --max-results 20
```

## Common Causes
1. **Environment not configured** — Consumer project needs environment for data access
2. **Grant failed** — Underlying permission grant (Glue/Redshift/RAM) failed
3. **Cross-account sharing** — RAM sharing not configured
4. **IAM permissions** — Environment role can't access shared data
5. **Subscription revoked** — Producer revoked the subscription

## Resolution
- Verify consumer project has an active environment
- Check subscription grant status for errors
- For cross-account: configure RAM sharing between accounts
- Verify environment IAM role has permissions for shared resources
- Check subscription status: APPROVED → GRANTED or GRANT_FAILED
- Contact producer if subscription was revoked


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Fulfillment grants actual data access (Glue/Redshift/RAM sharing) |
| Blast Radius | MEDIUM — Failed fulfillment blocks data access; incorrect fulfillment grants unauthorized access |
| Rollback Complexity | MEDIUM — Revoking access requires undoing Glue/Redshift/RAM grants |
| Compliance Scope | CRITICAL — Fulfillment is the enforcement point for data access governance |

## Escalation Conditions

- Escalate IMMEDIATELY to security if fulfillment grants access to unauthorized parties
- Escalate to data platform team if underlying grant mechanisms (Glue/RAM) fail
- Escalate to compliance if fulfillment failures prevent authorized data access

## Data Sensitivity

- **Subscription grants** create actual data access permissions — highest sensitivity
- **Cross-account RAM sharing** enables data access across account boundaries
- **Environment IAM roles** receive data access through fulfillment

## PROHIBITED Actions

- Do NOT manually grant data access outside the subscription workflow
- Do NOT ignore GRANT_FAILED status — it means data access was not properly established
- Do NOT configure cross-account sharing without security review

## Phase 3 — Rollback

- Revoke the subscription to remove data access grants
- Manually remove Glue/Redshift/RAM grants if automatic revocation fails
- Verify data access is fully revoked after rollback
- Document the incident for governance audit trail

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
