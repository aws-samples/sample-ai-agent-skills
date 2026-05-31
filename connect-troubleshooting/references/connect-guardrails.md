# Amazon Connect Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Connect issue.

## Guardrail 1: Lambda Timeout is 8 Seconds — Not Configurable
Lambda functions invoked from contact flows have a hard 8-second timeout enforced by Connect. This is independent of the Lambda function's own timeout setting. If your function needs more than 8 seconds, you must optimize it or use an async pattern.

## Guardrail 2: Contact Flow Logs Require Dual Enablement
Contact flow logging must be enabled at BOTH the instance level (instance attribute) AND on each individual contact flow (Set logging behavior block or flow setting). Enabling only one is insufficient — no logs will appear.

## Guardrail 3: DTMF Only in "Get Customer Input" Blocks
Only the "Get customer input" block captures DTMF tones. "Play prompt" blocks play audio but do NOT capture any input. If customers report pressing keys with no response, check which block type is being used.

## Guardrail 4: CTRs Have Up to 24-Hour Delivery Delay
Contact trace records delivered to the default Connect reporting can take up to 24 hours. For near-real-time CTR access, configure Kinesis data streaming. Don't assume missing CTRs are lost — they may just be delayed.

## Guardrail 5: CCP Requires WebRTC and Specific Network Access
The Contact Control Panel uses WebRTC for audio. It requires: ports 3478 UDP/TCP (TURN/STUN), access to *.awsapps.com and *.amazonaws.com domains, and a supported browser (Chrome or Firefox). Corporate firewalls and VPNs are the most common cause of CCP issues.

## Guardrail 6: Contact Attributes Have a 32 KB Total Limit
All contact attributes (system, user-defined, and external) combined cannot exceed 32 KB per contact. Exceeding this limit causes the contact flow to fail silently or drop attributes. Monitor attribute size in complex flows with many Lambda integrations.

## Guardrail 7: Agent "Available" vs Custom Routable States
Only agents in the "Available" state receive contacts by default. Custom agent statuses marked as "routable" only route contacts if the routing profile and queue configuration support it. Don't assume a custom status routes contacts without verifying the configuration.

## Guardrail 8: Phone Numbers Are Region-Bound
Phone numbers claimed in one AWS region cannot be used by Connect instances in another region. Porting a number to Connect requires it to be in the same region as the instance. Cross-region number sharing is not supported.

## Guardrail 9: Lex Bots Must Be in the Same Region
Lex bots used in Connect contact flows must be in the same AWS region as the Connect instance. Cross-region Lex integration is not supported. If the bot is in a different region, the "Get customer input" block will fail.

## Guardrail 10: Quick Connects Are Required for Transfers
Agents can only transfer contacts to destinations defined as quick connects. Without a quick connect entry, the transfer target won't appear in the CCP. This applies to agent-to-agent, agent-to-queue, and agent-to-external transfers.

## Guardrail 11: Contact Flow Definition Size Limit is 32 KB
The contact flow JSON definition cannot exceed 32 KB. Complex flows with many blocks, prompts, or branches can hit this limit. The flow will fail to save or publish. Split large flows into smaller sub-flows using "Transfer to flow" blocks.

## Guardrail 12: Outbound Caller ID Must Be a Claimed Number
Outbound calls require a caller ID number that is claimed in the Connect instance. You cannot set an arbitrary external number as the caller ID. The number must be a DID or toll-free number owned by the instance.
