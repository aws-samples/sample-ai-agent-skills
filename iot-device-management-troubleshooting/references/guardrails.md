# AWS IoT Device Management Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any IoT Device Management issue.

## Guardrail 1: Fleet Provisioning Has Two Distinct Flows
Claim-based provisioning uses a shared claim certificate for initial device bootstrap. Trusted-user provisioning uses an authorized user to register devices. These flows have different security models, IAM requirements, and certificate handling. Do not confuse the two.

## Guardrail 2: Bulk Registration Is All-or-Nothing Per Line
Bulk registration processes an S3 file line by line. A malformed JSON line causes that specific registration to fail. The overall bulk registration task may report partial success. Always check the error log S3 object for per-line failures.

## Guardrail 3: Job Execution Status Is Per-Device
A job has an overall status (IN_PROGRESS, COMPLETED) and per-device execution statuses (QUEUED, IN_PROGRESS, SUCCEEDED, FAILED, TIMED_OUT, REMOVED, CANCELED). The job-level status does not reflect individual device outcomes. Always check per-device execution status.

## Guardrail 4: Secure Tunneling Tokens Are Single-Use and Time-Limited
Tunnel access tokens expire after the configured lifetime (max 12 hours). Each token pair (source and destination) is single-use. Once a token is used or expires, a new tunnel must be opened. Tokens cannot be refreshed or extended.

## Guardrail 5: Dynamic Thing Groups Require Fleet Indexing
Dynamic thing groups use fleet indexing queries to determine membership. Fleet indexing must be enabled and configured BEFORE creating dynamic groups. If indexing is disabled, dynamic groups will have no members regardless of the query.

## Guardrail 6: Device Shadow Conflicts Use Last-Writer-Wins
When both the device and cloud application update the same shadow property simultaneously, the last write wins. There is no automatic conflict resolution or merge. Applications must implement their own conflict detection using shadow version numbers.

## Guardrail 7: OTA Updates Require Code Signing Configuration
OTA updates require a valid code-signing profile, correct S3 bucket permissions for the firmware file, and an IAM role with appropriate permissions. Missing any one of these causes the OTA update to fail. The signing profile must match the device's expected signing algorithm.

## Guardrail 8: Thing Types Are Immutable After Creation
Thing type properties (attribute names) cannot be modified after creation. To change attributes, you must deprecate the old thing type, create a new one, and migrate things. Deprecation has a 5-minute waiting period before deletion is allowed.

## Guardrail 9: Fleet Indexing Has Eventual Consistency
Fleet indexing updates are eventually consistent. After a thing is created, updated, or added to a group, the index may take several minutes to reflect the change. Do not rely on immediate index consistency for provisioning workflows.

## Guardrail 10: Job Timeouts Are Configurable at Two Levels
Jobs have two timeout settings: in-progress timeout (how long a device can work on a job) and notify-progress timeout (how long before a stalled job triggers notification). These are independent. A missing in-progress timeout means the job execution can run indefinitely.

## Guardrail 11: Secure Tunneling Requires Local Proxy on Both Ends
Secure tunneling establishes a WebSocket tunnel between source and destination. Both ends require a local proxy application running. The destination proxy is typically deployed via IoT jobs or pre-installed on the device. Without the proxy, the tunnel cannot carry SSH or other traffic.

## Guardrail 12: Provisioning Templates Use Specific IAM Roles
Fleet provisioning templates reference a provisioning role ARN that grants permissions to register things, create certificates, and attach policies. This role is assumed by the IoT service, not the device. Incorrect role permissions cause silent provisioning failures.
