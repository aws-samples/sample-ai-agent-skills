# Amazon Chime SDK Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Chime SDK issue.

## Guardrail 1: Chime SDK Has Multiple Separate Services
Chime SDK Meetings, Messaging, Voice, Identity, and Media Pipelines are separate services with different APIs and CLI namespaces. Do not mix commands between services (e.g., chime-sdk-meetings vs chime-sdk-messaging).

## Guardrail 2: Meetings Are Ephemeral Resources
Meetings are created on-demand and expire after all attendees leave (or after 5 minutes of inactivity). Meeting IDs cannot be reused. Each meeting session requires a new CreateMeeting call. Do not cache meeting IDs for reuse.

## Guardrail 3: Attendee Tokens Must Be Generated Server-Side
Attendee join tokens are generated via CreateAttendee API with AWS credentials. These tokens must be generated server-side and passed to the client. Client-side token generation would expose AWS credentials.

## Guardrail 4: Media Region Affects Latency
The media region determines where the meeting is hosted. Choose the region closest to the majority of participants. Available media regions may differ from standard AWS regions. Use the nearest media region for best quality.

## Guardrail 5: WebRTC Requires UDP Connectivity
Chime SDK meetings use WebRTC which requires UDP connectivity on ports 3478 and 443. Corporate firewalls blocking UDP cause audio/video failures. TURN relay is used as fallback but adds latency.

## Guardrail 6: SIP Media Applications Require Lambda Functions
SIP media applications use Lambda functions for call control logic. The Lambda function receives SIP events and returns actions. Without a properly configured Lambda, SIP calls fail immediately.

## Guardrail 7: Phone Numbers Must Be Provisioned Before Use
PSTN audio requires provisioned phone numbers. Numbers must be ordered and associated with SIP rules before they can receive calls. Provisioning can take minutes to hours depending on number type.

## Guardrail 8: Messaging Uses App Instance Identity Model
Chime SDK Messaging uses app instances and app instance users (not IAM users). App instance users must be created before they can send/receive messages. Each user needs a unique app instance user ARN.

## Guardrail 9: Channel Membership Controls Message Access
Users must be channel members to send/receive messages in a channel. Channel membership is managed via the Messaging API. Moderators can manage membership; regular members cannot add others.

## Guardrail 10: Voice Connectors Handle SIP Trunking
Voice connectors provide SIP trunking between your infrastructure and AWS. They require proper origination and termination settings. IP allowlisting and credential-based authentication are separate configuration steps.

## Guardrail 11: Media Pipelines Are Separate from Meetings
Media capture pipelines (recording, transcription) are created separately from meetings. They require the meeting ID and appropriate IAM permissions. Pipelines must be started after the meeting is created.

## Guardrail 12: SDK Version Compatibility Matters
Chime SDK client libraries (JS, iOS, Android) must be compatible with the server-side API version. Mixing incompatible versions causes subtle failures. Always use matching SDK versions across client and server.
