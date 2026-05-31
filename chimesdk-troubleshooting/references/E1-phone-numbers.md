# E1 — Chime SDK Phone Number Issues

## Symptoms
- Phone number provisioning fails
- Number not available for use
- Number assignment errors

## Triage Commands

```bash
aws chime-sdk-voice list-phone-numbers --max-results 20
aws chime-sdk-voice get-phone-number --phone-number-id <id>
aws chime-sdk-voice search-available-phone-numbers --area-code <code> --max-results 5
aws chime-sdk-voice list-phone-number-orders --max-results 10
```

## Common Causes

1. **No numbers available** — Area code or country has no available numbers
2. **Order pending** — Number provisioning takes time
3. **Number not assigned** — Provisioned but not assigned to SIP rule/voice connector
4. **Wrong product type** — SipMediaApplicationDialIn vs VoiceConnector
5. **Regulatory requirements** — Some countries require address verification

## Resolution

- Search available numbers: `aws chime-sdk-voice search-available-phone-numbers --area-code 206`
- Order number: `aws chime-sdk-voice create-phone-number-order --product-type SipMediaApplicationDialIn --e164-phone-numbers <number>`
- Check order status: `aws chime-sdk-voice get-phone-number-order --phone-number-order-id <id>`
- Assign to SIP rule after provisioning completes


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Search available numbers | GREEN |
| Order number with correct product type | YELLOW |
| Check order status | GREEN |
| Assign to SIP rule after provisioning completes | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Phone number provisioning details may contain business contact information and routing configurations

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If a phone number was ordered in error, release it: `aws chime-sdk-voice delete-phone-number --phone-number-id <id>`
2. If a number was assigned to the wrong SIP rule, reassign it to the correct rule
3. If the wrong product type was used, release the number and reorder with the correct type
4. Verify rollback by confirming `aws chime-sdk-voice list-phone-numbers` shows expected numbers

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
