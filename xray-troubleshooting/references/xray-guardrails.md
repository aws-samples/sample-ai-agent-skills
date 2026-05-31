# AWS X-Ray Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS X-Ray issue.

## Guardrail 1: Daemon Required for Non-Lambda Services
The X-Ray daemon must be running and reachable on UDP port 2000 for all services except Lambda. Lambda has built-in X-Ray integration. For ECS, run the daemon as a sidecar container in the same task definition. For EC2, run it as a system service. If the daemon is not running, segments are silently lost — no error is returned to the application.

## Guardrail 2: Sampling Rules Are Evaluated in Priority Order
Sampling rules are evaluated from lowest priority number to highest. The first matching rule determines the sampling rate. The default rule (priority 10000) catches everything not matched by custom rules. A higher-priority rule (lower number) with a 0% rate will suppress traces even if a lower-priority rule would sample them. Always check rule priorities.

## Guardrail 3: Segments vs Subsegments Are Different Concepts
Segments represent a service processing a request (created by the service itself). Subsegments represent downstream calls made by that service (AWS SDK calls, HTTP calls, SQL queries). Missing subsegments mean the SDK is not instrumented for those specific calls. You cannot have subsegments without a parent segment.

## Guardrail 4: Trace ID Propagation Requires Header Forwarding
The X-Amzn-Trace-Id header must be forwarded between services for traces to be connected. If a service does not propagate this header, downstream traces appear as separate, disconnected traces. API Gateway, ALB, and Lambda automatically propagate the header. Custom services must explicitly forward it.

## Guardrail 5: 64KB Segment Document Size Limit
Each segment document has a maximum size of 64KB. Traces with too many subsegments, large annotations, or extensive metadata will be truncated or rejected. Keep annotations to essential key-value pairs. Use metadata for detailed data but be mindful of size. Monitor for segment rejection errors.

## Guardrail 6: Trace Data Retained for 30 Days Only
X-Ray retains trace data for 30 days. After that, traces are permanently deleted and cannot be recovered. For long-term analysis, export trace data to S3 or use CloudWatch metrics derived from X-Ray data. Do not promise access to traces older than 30 days.

## Guardrail 7: Lambda X-Ray Integration Is Automatic but Requires Permissions
Lambda automatically creates segments for function invocations when X-Ray tracing is enabled. However, the function's execution role must have xray:PutTraceSegments and xray:PutTelemetryRecords permissions. Missing permissions cause silent trace loss. Downstream SDK calls still require X-Ray SDK instrumentation in the function code.

## Guardrail 8: ECS Requires Daemon as Sidecar Container
For ECS tasks, the X-Ray daemon must run as a sidecar container in the same task definition. Fargate tasks must use the sidecar approach. EC2 launch type can alternatively use a daemon running on the host instance. The application container must be able to reach the daemon on UDP port 2000.

## Guardrail 9: SDK Instrumentation Is Required for Downstream Visibility
Simply enabling X-Ray tracing on a service (API Gateway, Lambda) only captures that service's segment. To see downstream calls (DynamoDB, S3, HTTP), the application code must be instrumented with the X-Ray SDK or OpenTelemetry SDK. Without instrumentation, downstream calls appear as gaps in the trace.

## Guardrail 10: X-Ray SDK and OpenTelemetry SDK Are Different Approaches
The AWS X-Ray SDK is AWS-specific and tightly integrated with X-Ray. The OpenTelemetry SDK is vendor-neutral and can export to X-Ray via the ADOT collector. They use different instrumentation APIs. Do not mix both SDKs in the same application. Choose one approach and use it consistently.

## Guardrail 11: Service Map Time Range Defaults to 6 Hours
The X-Ray service map shows data from the last 6 hours by default. If you're investigating an issue from earlier, adjust the time range. The service map is generated from trace data — if traces were not sampled during a period, those services won't appear in the map for that time range.

## Guardrail 12: Sampling Statistics Show Actual vs Configured Rates
The get-sampling-statistic-summaries API shows actual sampling behavior, which may differ from configured rules. Reservoir size (guaranteed samples per second) and fixed rate (percentage after reservoir) work together. A rule with reservoir=1 and rate=0.05 samples 1 request/second guaranteed plus 5% of remaining requests.
