# AWS IoT Core Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS IoT Core issue.

## Guardrail 1: IoT Policies Are NOT IAM Policies
IoT policies are attached to X.509 certificates or Cognito identities, NOT to IAM users or roles. They use IoT-specific actions (iot:Connect, iot:Publish, iot:Subscribe, iot:Receive) and IoT resource ARNs. IAM policies control access to IoT management APIs (CreateThing, etc.). Do not confuse the two policy systems.

## Guardrail 2: Certificate Must Be ACTIVE and Properly Attached
A certificate must be in ACTIVE state to authenticate connections. It must be attached to both a thing (for identity) and an IoT policy (for authorization). INACTIVE, REVOKED, or PENDING_TRANSFER certificates will reject connections. Check certificate status with describe-certificate. Check attachments with list-thing-principals and list-attached-policies.

## Guardrail 3: MQTT Topic Wildcards Have Specific Semantics
The + wildcard matches exactly one topic level (e.g., sensor/+/data matches sensor/temp/data but not sensor/temp/room1/data). The # wildcard matches all remaining levels and must be the last character (e.g., sensor/# matches sensor/temp/data and sensor/temp/room1/data). Topic names are case-sensitive. Leading $ is reserved for system topics.

## Guardrail 4: Device Shadow Versioning Prevents Lost Updates
Each shadow update increments the version number. If you include a version in your update request, IoT Core rejects the update if the version doesn't match (optimistic locking). Without version checking, last-write-wins. Shadow documents have a maximum size of 8 KB for classic shadows. Named shadows allow multiple shadows per thing.

## Guardrail 5: Rule Engine SQL Has IoT-Specific Syntax
Rule SQL supports SELECT, FROM, WHERE with IoT-specific functions (topic(), timestamp(), encode(), etc.). The FROM clause specifies the MQTT topic filter, not a database table. Errors in SQL syntax cause silent rule failures — no error is returned to the publisher. Always test rules with the IoT console test feature before deploying.

## Guardrail 6: Message Broker Has Per-Account Throughput Limits
Publish requests are limited per second per account per region. Subscribe requests have separate limits. Exceeding limits causes throttling (HTTP 503 or MQTT PUBACK failure). Limits vary by region. Check CloudWatch metrics: PublishIn.ThrottledCount, Subscribe.ThrottledCount. Request increases through Service Quotas.

## Guardrail 7: Custom Authorizers Must Return Valid Policy Documents
Custom authorizers invoke a Lambda function that must return: isAuthenticated (boolean), principalId (string), policyDocuments (array of IoT policy documents), and optionally disconnectAfterInSeconds and refreshAfterInSeconds. Lambda timeout must be under 5 seconds. Invalid responses cause connection rejection.

## Guardrail 8: Thing Group Policies Apply to All Members
Policies attached to a thing group apply to all things in that group. Things can be in multiple groups and receive the union of all group policies. Dynamic thing groups use fleet indexing queries for automatic membership. Nested groups inherit parent group policies. Removing a thing from a group removes that group's policy access.

## Guardrail 9: Reserved MQTT Topics Have Special Behavior
Topics starting with $aws/ are reserved for IoT Core features (shadows, jobs, provisioning). $aws/things/<thing>/shadow/ topics are for device shadow operations. Publishing to reserved topics without proper permissions fails silently. The $aws/events/ topic provides lifecycle events. Do not use $ prefix for custom topics.

## Guardrail 10: MQTT QoS Levels Affect Message Delivery
IoT Core supports QoS 0 (at most once) and QoS 1 (at least once). QoS 2 is NOT supported. QoS 0 messages may be lost. QoS 1 messages are acknowledged but may be delivered more than once. The rule engine processes messages at QoS 0 regardless of the publish QoS. Choose QoS based on reliability requirements.

## Guardrail 11: Persistent Sessions Require Clean Session Flag
Setting cleanSession=false creates a persistent session. IoT Core stores subscriptions and queues QoS 1 messages while the device is offline. Messages are stored for up to 1 hour (configurable). The client ID must be consistent across connections. Persistent sessions have per-account limits on stored messages.

## Guardrail 12: Fleet Indexing Must Be Enabled for Advanced Queries
Fleet indexing is not enabled by default. It must be explicitly enabled for thing indexing, thing group indexing, or thing connectivity indexing. Dynamic thing groups require fleet indexing. Indexing has additional costs. Index updates are eventually consistent — there may be a delay between changes and query results.
