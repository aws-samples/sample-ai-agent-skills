# D1 — DataZone Subscription Request Issues

## Symptoms
- Subscription request fails
- Request stuck in pending
- Can't find asset to subscribe to

## Triage Commands

```bash
aws datazone list-subscriptions --domain-identifier <domain-id> --max-results 20
aws datazone list-subscription-requests --domain-identifier <domain-id> --max-results 20
aws datazone get-subscription-request-details --domain-identifier <domain-id> --identifier <request-id>
```

## Common Causes
1. **Asset not published** — Can only subscribe to published assets
2. **Not a project member** — Must be member of a consumer project
3. **Request already exists** — Duplicate subscription request
4. **Approval pending** — Request waiting for producer approval
5. **Auto-approval not configured** — Manual approval required

## Resolution
- Verify asset is published and visible in catalog
- Ensure user is member of a consumer project
- Check for existing subscription requests to same asset
- Contact asset owner for approval if request is pending
- Configure auto-approval policies for streamlined access


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Subscriptions grant access to data assets |
| Blast Radius | LOW — Subscription requests affect only the requesting project |
| Rollback Complexity | LOW — Subscriptions can be revoked |
| Compliance Scope | HIGH — Subscription approval is a data access governance control |

## Escalation Conditions

- Escalate to data owners if subscription requests are stuck in pending
- Escalate to compliance if auto-approval grants access to sensitive data without review

## Data Sensitivity

- **Subscription requests** reveal who is requesting access to what data
- **Approval decisions** are governance records — document for audit

## PROHIBITED Actions

- Do NOT configure auto-approval for sensitive or regulated data assets
- Do NOT approve subscription requests without verifying the requester's authorization
- Do NOT bypass the subscription workflow for data access

## Phase 3 — Rollback

- Reject incorrectly approved subscription requests
- Revoke granted subscriptions: revoke access through the subscription management API
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
