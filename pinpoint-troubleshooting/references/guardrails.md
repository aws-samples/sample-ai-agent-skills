# Amazon Pinpoint Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Pinpoint issue.

## Guardrail 1: Email Channel Requires SES Verified Identity
Pinpoint email channel uses SES under the hood. The sending identity (domain or email address) must be verified in SES before emails can be sent. Unverified identities cause silent delivery failures. Check verification status with `aws ses get-identity-verification-attributes`.

## Guardrail 2: SMS Requires Phone Number Provisioning
SMS messages require a provisioned origination number (long code, short code, or toll-free). Without a provisioned number, SMS sending fails. Use `aws pinpoint-sms-voice-v2 describe-phone-numbers` to check. Numbers must be provisioned in the target country.

## Guardrail 3: US SMS Traffic Requires 10DLC Registration
Sending SMS to US numbers using long codes requires 10DLC (10-Digit Long Code) registration. Unregistered traffic is filtered by carriers with no error returned to the sender. Registration involves company registration and campaign registration through the Pinpoint console.

## Guardrail 4: Campaign Scheduling Uses UTC by Default
Campaign schedules use UTC unless a timezone is explicitly specified. This causes campaigns to fire at unexpected local times. Always verify the timezone setting in campaign configuration. Recurring campaigns use the schedule's timezone for each execution.

## Guardrail 5: Segments Are Eventually Consistent
Dynamic segments based on endpoint attributes may not reflect recent endpoint updates immediately. There is a propagation delay. Import-based segments are available after the import job completes. Large segments (millions of endpoints) take longer to compute.

## Guardrail 6: Journey Maximum Duration Is 18 Months
Journeys have a maximum duration of 18 months (540 days). After this period, the journey stops accepting new participants. Active participants complete their current path. Plan journey lifecycle accordingly.

## Guardrail 7: Push Notifications Require Platform Credentials
Each push channel (APNs for iOS, FCM for Android, ADM for Kindle) requires platform-specific credentials configured in Pinpoint. APNs needs a signing key or certificate. FCM needs a server key or service account. Missing or expired credentials cause delivery failures.

## Guardrail 8: Event Streams Require Proper IAM Configuration
Pinpoint event streams to Kinesis Data Streams or Kinesis Data Firehose require an IAM role with write permissions. Missing permissions cause silent event loss — no error is returned to Pinpoint. Verify the role trust policy allows pinpoint.amazonaws.com.

## Guardrail 9: SMS Spending Limit Defaults to $1.00/Month
New accounts have a default SMS spending limit of $1.00 USD per month. This is quickly exhausted in production. Request a spending limit increase through AWS Support. Monitor spending with CloudWatch metrics.

## Guardrail 10: Endpoint Quotas Affect Campaign Reach
Each Pinpoint application has a default limit of 15 million endpoints. Endpoints that haven't been active in 30 days may be removed. Duplicate endpoints (same address, different endpoint IDs) inflate counts without increasing reach.

## Guardrail 11: Channel-Specific Opt-Out Is Automatic
Pinpoint automatically processes opt-out requests per channel. SMS STOP messages automatically opt out the number. Email unsubscribe links trigger opt-out. Opted-out endpoints are skipped during campaign delivery — they don't appear as failures.

## Guardrail 12: Sandbox Mode Limits Email Sending
New Pinpoint projects start in SES sandbox mode for email. Sandbox limits sending to verified email addresses only and caps daily volume. Request production access to send to any address. Check sandbox status in SES console.
