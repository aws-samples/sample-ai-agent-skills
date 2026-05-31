# AWS IoT Device Defender Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any IoT Device Defender issue.

## Guardrail 1: Audit and Detect Are Separate Features
Audit checks IoT account configuration against best practices (scheduled or on-demand). Detect monitors device behavior in real-time using security profiles. They have separate configurations, separate findings, and separate remediation workflows. Do not conflate them.

## Guardrail 2: ML Detect Requires Training Period
ML Detect models require a minimum 14-day training period with sufficient data before they can detect anomalies. During training, the model learns normal behavior patterns. No violations are generated during training. Do not assume ML Detect works immediately after configuration.

## Guardrail 3: Custom Metrics Require Device-Side Reporting
Custom metrics are NOT collected automatically by AWS. Devices must publish metrics to the reserved MQTT topic $aws/things/{thingName}/defender/metrics/json. If devices do not report metrics, no data is available for Detect to analyze. Verify device-side metric reporting before diagnosing Detect issues.

## Guardrail 4: Security Profiles Must Be Attached to Targets
Security profiles define expected behaviors but must be attached to thing groups or all things to be effective. An unattached security profile does not monitor any devices. Verify profile attachment before diagnosing missing violations.

## Guardrail 5: Alert Targets Are Per Security Profile
Alert targets (SNS topics) are configured per security profile, not globally. A security profile without an alert target will record violations but NOT send notifications. Check alert target configuration on the specific security profile.

## Guardrail 6: Audit Checks Have Specific Scopes
Each audit check examines a specific aspect of IoT configuration (e.g., CA certificate revocation, IoT policy permissiveness, logging configuration). Not all checks are enabled by default. Verify which checks are enabled before expecting specific findings.

## Guardrail 7: Mitigation Actions Are Pre-Defined Templates
Mitigation actions are pre-configured action templates (add thing to group, enable logging, publish to SNS, replace default policy version, update CA certificate, update device certificate). They are NOT custom Lambda functions. Do not suggest creating custom mitigation actions.

## Guardrail 8: Detect Violations Have Confidence Levels
Detect violations include confidence levels and statistical information. A single metric deviation may not indicate a real security issue. Review violation patterns over time rather than reacting to individual violations. ML Detect provides anomaly scores.

## Guardrail 9: Fleet Indexing Must Be Enabled for Aggregation
Device Defender violation data in fleet indexing requires fleet indexing to be enabled with Device Defender violations as an indexing source. Without this configuration, violation data is not queryable via fleet indexing.

## Guardrail 10: Dimensions Filter Security Profile Scope
Dimensions allow filtering which devices a security profile monitors based on topic filters. Misconfigured dimensions can cause a security profile to monitor no devices or the wrong devices. Verify dimension configuration when violations are missing.

## Guardrail 11: Audit Findings Are Point-in-Time
Audit findings represent the state at the time the audit ran. They do not update in real-time. A finding may be resolved between audit runs but still appear in the last audit results. Run a new audit to verify current state.

## Guardrail 12: SNS Topic Permissions Must Allow IoT
The SNS topic used as an alert target must have a resource policy allowing the IoT service principal (iot.amazonaws.com) to publish. Without this permission, alert notifications silently fail. Check SNS topic policy, not just IAM permissions.
