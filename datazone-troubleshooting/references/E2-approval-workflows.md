# E2 — DataZone Approval Workflow Issues

## Symptoms
- Approval requests not reaching approvers
- Auto-approval not working
- Approval stuck in pending

## Triage Commands

```bash
aws datazone list-subscription-requests --domain-identifier <domain-id> --status PENDING --max-results 20
aws datazone get-subscription-request-details --domain-identifier <domain-id> --identifier <request-id>
```

## Common Causes
1. **No approver configured** — Project has no designated approver
2. **Approver not notified** — Notification not reaching approver
3. **Auto-approval misconfigured** — Policy conditions not matching
4. **Approver left organization** — Approver no longer has access
5. **Workflow timeout** — Request expired without action

## Resolution
- Verify project has designated approvers (project owners)
- Check notification configuration for approvers
- Review auto-approval policy conditions and scope
- Update approver if previous one is unavailable
- Approvers can approve/reject via portal or API: `aws datazone accept-subscription-request` or `aws datazone reject-subscription-request`


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Approval workflows control data access decisions |
| Blast Radius | MEDIUM — Broken workflows block data access for consumers |
| Rollback Complexity | LOW — Approval settings can be updated |
| Compliance Scope | HIGH — Approval workflows are a key governance control |

## Escalation Conditions

- Escalate to project owners if approval requests are stuck without an approver
- Escalate to compliance if auto-approval is misconfigured for sensitive data
- Escalate to management if approval workflow failures block business-critical data access

## Data Sensitivity

- **Approval decisions** are governance records — preserve for audit
- **Approver identities** and their decisions should be logged

## PROHIBITED Actions

- Do NOT configure auto-approval for data classified as sensitive or regulated
- Do NOT approve requests without verifying the requester's business justification
- Do NOT remove approvers from projects without designating replacements

## Phase 3 — Rollback

- Reassign approver roles if the previous approver is unavailable
- Revert auto-approval policy changes if they were misconfigured
- Manually process stuck approval requests
- Document approval workflow changes for governance records

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
