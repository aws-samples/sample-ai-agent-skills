---
title: "B2 — Sync Client Issues"
description: "Diagnose WorkDocs sync client problems"
status: active
severity: MEDIUM
triggers: ["sync not working", "sync client error", "files not syncing", "sync stuck"]
owner: devops-agent
objective: "Resolve WorkDocs desktop sync client issues including connectivity, conflicts, and local storage problems"
context: "The WorkDocs sync client is a desktop application. Most sync issues are client-side: network, disk space, OS permissions, or client version."
---

## Phase 1 — Triage

MUST:
- Verify user account is active: `aws workdocs describe-users --organization-id <org-id> --query 'Users[?Username==`<username>`].{Status:Status,Storage:StorageRule}'`
- Check if the issue is client-side or service-side by testing web client access
- Verify network connectivity to WorkDocs endpoint from the client machine
- Check local disk space on the client machine

SHOULD:
- Check sync client version (recommend latest version)
- Review sync client logs for error messages
- Verify no file conflicts exist in the sync folder

MAY:
- Check if antivirus or firewall is blocking the sync client
- Verify proxy settings if the client is behind a corporate proxy

## Phase 2 — Remediate

MUST:
- Update sync client to the latest version
- Ensure sufficient local disk space for synced content
- Resolve file conflicts by choosing the correct version

SHOULD:
- Clear sync client cache and restart the application
- Verify proxy and firewall settings allow HTTPS to WorkDocs endpoints
- Re-authenticate the sync client if credentials have expired

MAY:
- Reinstall the sync client if persistent issues after cache clear
- Configure selective sync to reduce local storage requirements

## Safety Ratings

```
safety_ratings:
  - "workdocs describe-users: GREEN — read-only user inspection"
  - "sync client restart: GREEN — restarts local application"
  - "sync client cache clear: YELLOW — removes local cache, triggers re-sync"
  - "sync client reinstall: YELLOW — removes and reinstalls local application"
```

## Escalation Conditions

- Sync client fails for all users in the organization
- Sync client crashes repeatedly after reinstallation
- File conflicts cannot be resolved and data loss is possible
- Sync client cannot authenticate despite valid credentials

## Data Sensitivity

- `aws workdocs describe-users`: MEDIUM — reveals user identities
- Sync client logs: MEDIUM — may contain file names and user activity
- Local sync folder: HIGH — contains copies of WorkDocs documents

## Prohibited Actions

- NEVER delete the local sync folder without backing up unsynced changes
- NEVER force-sync by deleting conflict files without user review
- NEVER share sync client logs containing file names or user data externally
- NEVER downgrade the sync client version as a troubleshooting step

## Phase 3 — Rollback

1. If sync client cache was cleared, wait for re-sync to complete before making further changes
2. If selective sync settings were changed, restore original sync folder selections
3. If the sync client was reinstalled, reconfigure sync folder location and settings
4. Restore any files from the WorkDocs web client if local copies were lost
5. Document the sync client version and configuration for future reference

## Common Issues

- symptoms: "Sync client shows 'Unable to connect' error"
  diagnosis: "Network connectivity issue or proxy blocking HTTPS to WorkDocs."
  resolution: "Check network/proxy settings. Ensure HTTPS traffic to *.workdocs.aws is allowed."

- symptoms: "Files show conflict icons in sync folder"
  diagnosis: "Same file was modified both locally and on the server."
  resolution: "Open conflict resolution in sync client. Choose the correct version for each conflicted file."

## Output Format

```yaml
root_cause: "sync_client — <specific_cause>"
evidence:
  - type: sync_issue
    content: "<client version, error details, network status>"
severity: MEDIUM
mitigation:
  immediate: "Fix client-side issue"
  long_term: "Keep sync client updated and monitor disk space"
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
