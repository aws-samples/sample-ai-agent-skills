# Amazon SNS Advanced Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any SNS issue.

## Guardrail 1: Delivery Status Logging Is NOT Enabled by Default
Delivery status logging must be explicitly enabled per protocol (HTTP/S, Lambda, SQS, SMS, Application/Mobile Push). Without it, you have no visibility into delivery failures. Enable it via topic attributes: set the appropriate *SuccessFeedbackRoleArn and *FailureFeedbackRoleArn attributes. Logs go to CloudWatch Logs.

## Guardrail 2: SMS Sandbox Limits Sending to Verified Numbers Only
New AWS accounts are in the SMS sandbox by default. In sandbox mode, you can only send SMS to verified phone numbers (up to 10). To send to any number, you must request production access. Do not diagnose SMS delivery failures without first checking sandbox status.

## Guardrail 3: SMS Opt-Out Silently Drops Messages
When a recipient opts out by replying STOP, subsequent messages to that number are silently dropped — no error is returned. Always check the opt-out list with `check-if-phone-number-is-opted-out` before diagnosing SMS delivery failures. Opt-out is per-number, not per-topic.

## Guardrail 4: Filter Policy Scope Defaults to MessageAttributes
By default, subscription filter policies match against message attributes, NOT the message body. To filter on message body content, you must set FilterPolicyScope to MessageBody on the subscription. Do not assume body-based filtering works without checking the scope setting.

## Guardrail 5: FIFO Topics Require .fifo Suffix
FIFO topic names must end with .fifo (e.g., my-topic.fifo). Standard topics cannot be converted to FIFO. FIFO topics only support SQS FIFO queues as subscriptions (not standard SQS, HTTP, Lambda, etc.). Do not suggest converting a standard topic to FIFO.

## Guardrail 6: DLQs Are Configured Per-Subscription, Not Per-Topic
Dead-letter queues in SNS are set on individual subscriptions, not on the topic. Each subscription can have its own DLQ (SQS queue). The DLQ must be in the same Region and account as the subscription. Do not suggest configuring a DLQ on the topic itself.

## Guardrail 7: Mobile Push Credentials Must Match the Platform
Platform applications require platform-specific credentials: APNs requires a certificate or token-based auth key (p8) for iOS. FCM requires a server key or service account JSON for Android. Using wrong credentials causes all push notifications to fail. Credentials expire and must be rotated.

## Guardrail 8: Retry Policies Vary by Protocol
SNS retry policies differ by protocol: HTTP/S retries for up to 23 days with exponential backoff (3 immediate, then backoff). Lambda retries 3 times immediately. SQS has no retries (single immediate delivery). SMS has no retries. Do not assume uniform retry behavior across protocols.

## Guardrail 9: Message Size Limit Is 256 KB
SNS messages have a 256 KB size limit for both standard and FIFO topics. This includes the message body and all message attributes. For larger payloads, store the data in S3 and include a reference in the SNS message. Do not suggest sending messages larger than 256 KB.

## Guardrail 10: Cross-Account Subscriptions Require Confirmation
When an account subscribes to a topic in another account, the subscription must be confirmed by the endpoint owner. Unconfirmed subscriptions do not receive messages. The topic policy must allow the subscribing account to call sns:Subscribe. Check subscription confirmation status.

## Guardrail 11: Raw Message Delivery Changes the Message Format
When RawMessageDelivery is enabled on a subscription, SNS sends the raw message without the SNS JSON envelope. This affects SQS and HTTP/S subscriptions. If the subscriber expects the SNS envelope (with Message, MessageId, etc.), raw delivery will break parsing. Always check this setting.

## Guardrail 12: Subscription Confirmation Tokens Expire After 3 Days
HTTP/S subscription confirmation tokens expire after 3 days. If the endpoint doesn't confirm within this window, the subscription remains in PendingConfirmation state and must be re-created. Do not suggest confirming an expired subscription — delete and recreate it.
