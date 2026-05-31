# CloudWatch Internet Monitor Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Internet Monitor issue.

## Guardrail 1: Passive Traffic Analysis Not Synthetic Monitoring
Internet Monitor uses passive traffic analysis of existing application traffic. It does NOT send synthetic probes or active health checks. If there is no traffic flowing through monitored resources, Internet Monitor cannot generate scores or detect issues.

## Guardrail 2: Monitors Require At Least One Resource
A monitor must have at least one monitored resource (CloudFront distribution or VPC). A monitor with no resources will remain INACTIVE and produce no data. Verify resources are attached before investigating missing data.

## Guardrail 3: Health Events Reflect Observed Degradation
Health events are generated when observed performance or availability drops below configured thresholds. They reflect actual traffic degradation, not predictions. Do not claim Internet Monitor predicts future outages.

## Guardrail 4: Scores Are Per City-Network Pair
Availability and performance scores are calculated per city-network (ASN) pair. A degradation in one city-network may not significantly affect the overall monitor score. Always drill into city-network level data for accurate diagnosis.

## Guardrail 5: MaxCityNetworksToMonitor Affects Coverage and Cost
The MaxCityNetworksToMonitor parameter controls how many city-networks are tracked. Lower values reduce cost but may miss issues in less-trafficked locations. The default is 500,000. Do not assume all city-networks are monitored if this value is reduced.

## Guardrail 6: Internet Monitor Is Regional
Each monitor operates in the region where it is created. It monitors resources (CloudFront distributions, VPCs) in that region. CloudFront distributions are global but must be added to a monitor in a specific region.

## Guardrail 7: Log Delivery Is Not Enabled by Default
S3 and CloudWatch Logs delivery must be explicitly configured on the monitor. They are not enabled by default. Missing logs does not indicate a failure — verify delivery is configured before investigating.

## Guardrail 8: Internet Events Are Global Not Monitor-Specific
Internet events (list-internet-events) are global AWS-observed internet disruptions. They are NOT specific to your monitor or application. Do not conflate global internet events with your application's health events.

## Guardrail 9: Processing Status Indicates Data Pipeline Health
A monitor's ProcessingStatus field indicates whether data is being processed. COLLECTING_DATA means the monitor is gathering initial data. OK means normal operation. INSUFFICIENT_DATA means not enough traffic for scoring.

## Guardrail 10: Availability Score Is Not Uptime
The availability score measures the percentage of traffic that completed successfully, not server uptime. A 95% availability score means 5% of observed traffic experienced failures, not that the service was down 5% of the time.

## Guardrail 11: CloudFront and VPC Resources Have Different Metrics
CloudFront distributions provide both performance and availability metrics. VPC resources provide availability metrics but performance metrics may be limited. Do not expect identical metric coverage across resource types.

## Guardrail 12: Query API Has Time Range Limitations
The start-query API for traffic insights has time range limitations. Queries cannot span more than the data retention period. Historical data availability depends on how long the monitor has been active.
