# F1 — Chime SDK Identity Management Issues

## Symptoms
- App instance creation fails
- App instance user errors
- Authentication/authorization failures

## Triage Commands

```bash
aws chime-sdk-identity list-app-instances --max-results 20
aws chime-sdk-identity describe-app-instance --app-instance-arn <arn>
aws chime-sdk-identity list-app-instance-users --app-instance-arn <arn> --max-results 20
aws chime-sdk-identity describe-app-instance-user --app-instance-user-arn <user-arn>
```

## Common Causes

1. **App instance limit** — Service quota on app instances per account
2. **Duplicate user ID** — App instance user IDs must be unique within instance
3. **User not found** — User ARN doesn't exist or was deleted
4. **Admin not configured** — App instance admin needed for some operations
5. **Wrong ARN format** — App instance user ARN has specific format

## Resolution

- Create app instance: `aws chime-sdk-identity create-app-instance --name <name>`
- Create user: `aws chime-sdk-identity create-app-instance-user --app-instance-arn <arn> --app-instance-user-id <id> --name <display-name>`
- Promote admin: `aws chime-sdk-identity create-app-instance-admin --app-instance-admin-arn <user-arn> --app-instance-arn <arn>`
- User ARN format: `arn:aws:chime:<region>:<account>:app-instance/<instance-id>/user/<user-id>`


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create app instance | YELLOW |
| Create user in app instance | GREEN |
| Promote admin | YELLOW |
| Use correct user ARN format | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Identity management configurations contain user identities and admin privileges

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If an admin was promoted in error, remove admin: `aws chime-sdk-identity delete-app-instance-admin --app-instance-admin-arn <user-arn> --app-instance-arn <arn>`
2. If users were created incorrectly, delete them: `aws chime-sdk-identity delete-app-instance-user --app-instance-user-arn <arn>`
3. If an app instance was created in error, delete it (cascades to all users): `aws chime-sdk-identity delete-app-instance --app-instance-arn <arn>`
4. Verify rollback by confirming `aws chime-sdk-identity list-app-instance-users` shows expected users

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
