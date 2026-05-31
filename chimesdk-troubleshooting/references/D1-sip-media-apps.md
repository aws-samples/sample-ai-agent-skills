# D1 — Chime SDK SIP Media Application Issues

## Symptoms
- SIP media application creation fails
- Incoming calls dropped immediately
- Lambda function not invoked

## Triage Commands

```bash
aws chime-sdk-voice list-sip-media-applications --max-results 20
aws chime-sdk-voice get-sip-media-application --sip-media-application-id <id>
aws chime-sdk-voice list-sip-rules --max-results 20
aws lambda get-function --function-name <lambda-name>
```

## Common Causes

1. **Lambda not configured** — SIP media app must point to valid Lambda ARN
2. **Lambda permissions** — Lambda needs resource-based policy for Chime SDK
3. **Lambda timeout** — Function must respond within 10 seconds
4. **SIP rule not created** — Phone number needs SIP rule to route to app
5. **Lambda response format** — Must return valid SIP media application actions

## Resolution

- Create Lambda with Chime SDK SIP media application handler
- Add Lambda permission: `aws lambda add-permission --function-name <name> --statement-id chime --action lambda:InvokeFunction --principal voiceconnector.chime.amazonaws.com`
- Create SIP media app: `aws chime-sdk-voice create-sip-media-application --aws-region <region> --endpoints LambdaArn=<arn>`
- Create SIP rule to route calls to the app
- Test Lambda independently with sample SIP events


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create Lambda with Chime SDK SIP media application handler | YELLOW |
| Add Lambda permission for voiceconnector.chime.amazonaws.com | YELLOW |
| Create SIP media app with Lambda endpoint | YELLOW |
| Create SIP rule to route calls to the app | YELLOW |
| Test Lambda independently with sample SIP events | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- SIP media applications handle voice call data and routing information

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If a SIP rule was created in error, delete it: `aws chime-sdk-voice delete-sip-rule --sip-rule-id <id>`
2. If a SIP media application was created in error, delete it: `aws chime-sdk-voice delete-sip-media-application --sip-media-application-id <id>`
3. If Lambda permissions were added, remove them: `aws lambda remove-permission --function-name <name> --statement-id chime`
4. If the Lambda function was modified, redeploy the previous version
5. Verify rollback by confirming `aws chime-sdk-voice list-sip-media-applications` shows expected state

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
