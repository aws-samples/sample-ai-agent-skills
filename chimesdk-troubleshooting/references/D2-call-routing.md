# D2 — Chime SDK Call Routing Issues

## Symptoms
- Calls not reaching SIP media application
- Wrong SIP rule matching
- Call routing failures

## Triage Commands

```bash
aws chime-sdk-voice list-sip-rules --max-results 20
aws chime-sdk-voice get-sip-rule --sip-rule-id <rule-id>
aws chime-sdk-voice list-phone-numbers --max-results 20
```

## Common Causes

1. **No SIP rule for number** — Phone number not associated with a SIP rule
2. **SIP rule disabled** — Rule exists but is disabled
3. **Priority conflict** — Multiple rules matching, wrong priority order
4. **Phone number not provisioned** — Number not yet active
5. **Trigger type mismatch** — ToPhoneNumber vs RequestUriHostname

## Resolution

- Create SIP rule: `aws chime-sdk-voice create-sip-rule --name <name> --trigger-type ToPhoneNumber --trigger-value <phone-number> --target-applications SipMediaApplicationId=<app-id>,Priority=1,AwsRegion=<region>`
- Verify rule is enabled: `Disabled: false`
- Check phone number status: must be in `Assigned` state
- For multiple apps, set priority (lower number = higher priority)


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create SIP rule for phone number routing | YELLOW |
| Verify rule is enabled | GREEN |
| Check phone number status is Assigned | GREEN |
| Set priority for multiple apps | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Call routing configurations contain phone numbers and routing logic for voice communications

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If a SIP rule was created or modified, restore the previous rule configuration or delete the new rule
2. If rule priority was changed, restore the previous priority order
3. If phone number assignment was changed, reassign to the previous SIP rule
4. Verify rollback by confirming `aws chime-sdk-voice list-sip-rules` shows expected routing

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
