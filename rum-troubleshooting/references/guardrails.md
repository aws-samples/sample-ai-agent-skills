# CloudWatch RUM Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any RUM issue.

## Guardrail 1: JavaScript Snippet Is Required for Data Collection
RUM collects data via a JavaScript snippet embedded in the web application. Without the snippet correctly installed and loaded, no data is collected. Always verify snippet installation before investigating missing data.

## Guardrail 2: Cognito Identity Pool Is Required
RUM uses Amazon Cognito Identity pools to authenticate browser clients. The identity pool must allow unauthenticated identities and have an IAM role with rum:PutRumEvents permission. Authentication failures prevent data collection.

## Guardrail 3: RUM Collects Real User Data Not Synthetic
RUM monitors real user browser sessions. It does NOT send synthetic probes or simulate user interactions. If no real users visit the application, no data is collected. Do not confuse RUM with CloudWatch Synthetics.

## Guardrail 4: Session Sample Rate Controls Data Volume
The session sample rate (0.0 to 1.0) controls what percentage of user sessions are recorded. A rate of 0.1 means only 10% of sessions are captured. Missing data may be due to sampling, not a collection failure.

## Guardrail 5: Custom Events Require Explicit Configuration
Custom events are not collected by default. They must be explicitly enabled in the app monitor configuration and the application must call the RUM web client API to record them. Missing custom events usually means they were not configured.

## Guardrail 6: Session Replay Is Optional and Costly
Session replay captures DOM changes and user interactions for playback. It is NOT enabled by default and significantly increases data volume and cost. Do not assume session replay is active without checking the configuration.

## Guardrail 7: CloudWatch Metric Export Is Not Automatic
RUM data is not automatically available as CloudWatch metrics. Metric export must be explicitly configured using put-rum-metrics-destination and batch-create-rum-metric-definitions. Missing CloudWatch metrics usually means export is not configured.

## Guardrail 8: RUM Is Regional
Each app monitor operates in the region where it is created. The JavaScript snippet sends data to that specific region. Cross-region data collection is not supported. Verify the app monitor region matches the expected data destination.

## Guardrail 9: Domain Configuration Affects Data Collection
The app monitor's allowed domains configuration controls which domains can send data. If the application domain is not in the allowed list, the snippet will not send data. Verify domain configuration matches the application's actual domain.

## Guardrail 10: Browser Compatibility Affects Coverage
RUM relies on browser APIs (Performance API, Navigation Timing, etc.). Older browsers may not support all features. Missing data from specific browsers may be due to compatibility, not a collection failure.

## Guardrail 11: Content Security Policy Can Block RUM
If the web application has a Content Security Policy (CSP), it must allow the RUM JavaScript snippet to load and send data to the Cognito and RUM endpoints. CSP violations silently block data collection.

## Guardrail 12: Data Retention Has Limits
RUM data retention is limited. Historical data beyond the retention period is not available. Do not assume all historical data is accessible. Check the data retention configuration for the app monitor.
