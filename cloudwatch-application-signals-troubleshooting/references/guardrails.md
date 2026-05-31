# CloudWatch Application Signals Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Application Signals issue.

## Guardrail 1: Two Agents Required for Full Functionality
Application Signals requires BOTH the CloudWatch agent AND the AWS Distro for OpenTelemetry (ADOT) auto-instrumentation agent. The CloudWatch agent collects and forwards metrics. The ADOT agent provides auto-instrumentation for traces and service metadata. Missing either agent results in incomplete data.

## Guardrail 2: Insufficient Data Is Not the Same as Healthy
When an SLO shows "Insufficient Data," it means metrics are not flowing — not that the service is healthy. This typically indicates an instrumentation or metric collection problem. Do not interpret insufficient data as a passing SLO.

## Guardrail 3: Service Discovery Depends on Instrumentation Attributes
Services appear in Application Signals only when the ADOT agent injects the correct resource attributes (service.name, deployment.environment, telemetry.sdk.language). Missing or incorrect attributes prevent service discovery. Do not assume services auto-discover.

## Guardrail 4: SLO Evaluation Uses Rolling Windows
SLOs evaluate over rolling time windows (e.g., 30 days). A brief outage may not immediately breach the SLO if the error budget is sufficient. Conversely, a service may breach its SLO due to accumulated small degradations. Always check the remaining error budget.

## Guardrail 5: Trace Correlation Requires X-Ray Integration
Application Signals correlates traces with metrics and logs using X-Ray trace IDs. If X-Ray tracing is not enabled or trace IDs are not propagated, correlation will fail. Application Signals and X-Ray are complementary — both must be configured.

## Guardrail 6: EKS Integration Requires Specific Addon Versions
The CloudWatch Observability EKS addon (amazon-cloudwatch-observability) must be at a compatible version. Older addon versions may not support Application Signals features. Verify addon version before troubleshooting feature gaps.

## Guardrail 7: EC2 Integration Is Manual
Unlike EKS where the addon handles setup, EC2 integration requires manual installation and configuration of both the CloudWatch agent and ADOT auto-instrumentation agent. There is no automatic setup for EC2 instances.

## Guardrail 8: Metrics Namespace Is AWS/ApplicationSignals
Application Signals metrics are published to the AWS/ApplicationSignals namespace. Do not look for them in custom namespaces or the default AWS/CloudWatch namespace. Missing metrics in this namespace indicate instrumentation issues.

## Guardrail 9: Service Dependencies Are Inferred from Traces
Service dependency maps are built from distributed trace data. If tracing is incomplete or trace context is not propagated between services, dependency maps will be incomplete. Missing dependencies usually indicate trace propagation issues.

## Guardrail 10: Application Signals Is Regional
Application Signals data is regional. Services running in different regions appear as separate services. Cross-region service dependencies may not be fully mapped. Ensure you are querying the correct region.

## Guardrail 11: Supported Languages Have Specific Requirements
Auto-instrumentation supports Java, Python, .NET, and Node.js. Each language has specific ADOT agent versions and configuration requirements. Unsupported languages require manual OpenTelemetry instrumentation.

## Guardrail 12: SLO Alarms Are Separate from SLO Definitions
Creating an SLO does not automatically create CloudWatch alarms. Alarms must be configured separately to notify on SLO breaches or error budget consumption. Do not assume SLO creation includes alerting.
