# C1 — Chime SDK Channel Messaging Issues

## Symptoms
- Channel creation fails
- Can't send messages to channel
- Channel not found

## Triage Commands

```bash
aws chime-sdk-identity list-app-instances --max-results 20
aws chime-sdk-messaging list-channels --app-instance-arn <app-instance-arn>
aws chime-sdk-messaging describe-channel --channel-arn <channel-arn> --app-instance-user-arn <user-arn>
aws chime-sdk-messaging list-channel-memberships --channel-arn <channel-arn>
```

## Common Causes

1. **No app instance** — Must create app instance first
2. **No app instance user** — Users must be created in the app instance
3. **Not a channel member** — Must be member to send messages
4. **Wrong app instance user ARN** — Using IAM ARN instead of app instance user ARN
5. **Channel privacy** — PRIVATE channels require explicit membership

## Resolution

- Create app instance: `aws chime-sdk-identity create-app-instance --name <name>`
- Create user: `aws chime-sdk-identity create-app-instance-user --app-instance-arn <arn> --app-instance-user-id <id>`
- Create channel: `aws chime-sdk-messaging create-channel --app-instance-arn <arn> --name <name>`
- Add member: `aws chime-sdk-messaging create-channel-membership --channel-arn <arn> --member-arn <user-arn> --type DEFAULT`


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create app instance | YELLOW |
| Create user in the app instance | GREEN |
| Create channel | GREEN |
| Add member to channel | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Channel messages may contain sensitive business communications and participant data

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If a channel was created in error, delete it: `aws chime-sdk-messaging delete-channel --channel-arn <arn>`
2. If users were created incorrectly, delete them: `aws chime-sdk-identity delete-app-instance-user --app-instance-user-arn <arn>`
3. If memberships were added in error, remove them: `aws chime-sdk-messaging delete-channel-membership --channel-arn <arn> --member-arn <user-arn>`
4. If an app instance was created in error, delete it (this deletes all users and channels): `aws chime-sdk-identity delete-app-instance --app-instance-arn <arn>`
5. Verify rollback by confirming `aws chime-sdk-messaging list-channels` shows expected state

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
