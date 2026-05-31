# Amazon Lex Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Lex issue.

## Guardrail 1: Bot Must Be Built After Every Change
Any change to intents, slots, utterances, or slot types requires a bot build before changes take effect. The build trains the NLU model. Without building, the bot uses the previous model. Check build status with `aws lexv2-models describe-bot-locale`.

## Guardrail 2: V2 and V1 APIs Are Completely Separate
Lex V2 uses bot-id and bot-alias-id (UUIDs). Lex V1 uses bot-name and alias-name (strings). V2 bots cannot be managed with V1 APIs and vice versa. V2 is the current recommended API. V1 is in maintenance mode.

## Guardrail 3: Slot Types Are Locale-Specific
Built-in slot types (AMAZON.Date, AMAZON.Number, etc.) vary by locale. Not all built-in types are available in all locales. Custom slot types must be created separately for each locale. Slot type values are language-specific.

## Guardrail 4: Lambda Fulfillment Has 30-Second Timeout
Lambda functions used for fulfillment or code hooks have a maximum timeout of 30 seconds in Lex. Even if the Lambda function timeout is set higher, Lex enforces its own 30-second limit. Long operations must be handled asynchronously.

## Guardrail 5: Conversation Logs Require IAM Role Configuration
Conversation logs (text and audio) require an IAM role with permissions to write to CloudWatch Logs (text) or S3 (audio). The role must have a trust policy allowing lexv2.amazonaws.com. Missing permissions cause silent log loss — no error returned to the user.

## Guardrail 6: Bot Aliases Point to Specific Versions
Aliases map to specific bot versions. The DRAFT version is mutable and for testing. Production traffic should use numbered versions via aliases. Updating an alias to a new version is atomic. The TestBotAlias always points to DRAFT.

## Guardrail 7: Streaming Requires WebSocket or HTTP/2
Streaming conversations (StartConversation API) use bidirectional event streams over HTTP/2. This is different from the synchronous RecognizeText/RecognizeUtterance APIs. Not all SDKs support streaming equally. WebSocket connections have idle timeout limits.

## Guardrail 8: FallbackIntent Is Required for Graceful Handling
Every locale should have a FallbackIntent to handle unrecognized utterances. Without it, unrecognized input may return errors. The FallbackIntent can trigger fulfillment Lambda for custom handling. It fires after the NLU confidence threshold is not met.

## Guardrail 9: NLU Confidence Threshold Affects Intent Matching
Lex V2 uses confidence scores for intent matching. The nluIntentConfidenceThreshold (default 0.4) determines the minimum score for a match. Setting too high causes excessive fallback. Setting too low causes mismatches. Tune based on testing.

## Guardrail 10: Session State Persists Across Turns
Lex maintains session state across conversation turns. Session attributes, slot values, and dialog state persist until the session expires or is explicitly cleared. Session TTL is configurable (default 5 minutes of idle time). Stale sessions can cause unexpected behavior.

## Guardrail 11: Bot Locale Must Be Built Before Bot Version
Building happens at the locale level. All locales must be built before creating a bot version. A bot version snapshots all locale builds. If a locale build fails, the bot version cannot be created.

## Guardrail 12: Multi-Turn Conversations Require Dialog Management
Slot elicitation, confirmation prompts, and follow-up prompts are part of dialog management. The bot manages conversation flow based on slot priorities and required/optional settings. Misconfigured dialog flow causes the bot to skip slots or ask repeatedly.
