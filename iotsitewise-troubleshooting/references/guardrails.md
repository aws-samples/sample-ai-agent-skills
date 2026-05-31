# AWS IoT SiteWise Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS IoT SiteWise issue.

## Guardrail 1: Asset Models Define Structure, Assets Are Instances
Asset models define property types, hierarchies, and formulas. Assets are instances of models. Changing a model affects all assets of that model. Models must be in ACTIVE state before creating assets. Model updates propagate asynchronously to all assets.

## Guardrail 2: Three Property Types Have Different Semantics
Measurements receive raw data from equipment. Transforms compute values from other properties in near real-time using formulas. Metrics aggregate data over time windows (1m, 5m, 15m, 1h, 1d). Each type has different ingestion, computation, and query behavior. Never confuse them.

## Guardrail 3: Transform Formulas Must Be Valid
Transforms use a formula language supporting math (+, -, *, /), comparison (>, <, ==), logical (and, or, not), and built-in functions (abs, ceil, floor, log, etc.). Formulas can reference other properties by variable name. Circular references are rejected. Division by zero returns undefined.

## Guardrail 4: Metric Time Windows Are Fixed
Metrics aggregate over predefined time windows. The window determines computation frequency. Metrics are computed after the window closes, not in real-time. Late-arriving data within the late data ingestion window is included. Metrics can reference child asset properties for hierarchical aggregation.

## Guardrail 5: Gateways Run on Greengrass V2
SiteWise gateways are Greengrass V2 components. The gateway device must have Greengrass V2 nucleus installed. Gateway capability configurations define OPC-UA, Modbus, or Ethernet/IP data sources. Gateway status and connectivity depend on Greengrass health.

## Guardrail 6: Portals Require IAM Identity Center
SiteWise Monitor portals authenticate users via IAM Identity Center (formerly SSO). IAM users cannot directly access portals. Portal administrators assign users to projects. Users can only see assets within their assigned projects. Portal creation requires IAM Identity Center to be enabled.

## Guardrail 7: Data Ingestion Has Rate Limits
BatchPutAssetPropertyValue allows up to 10 entries per request. Each property has a per-second ingestion rate limit. Account-level limits apply across all properties. Exceeding limits returns throttling errors. Timestamps must be within the allowed range (not too far in the past or future).

## Guardrail 8: Timestamps Must Be Within Allowed Range
Property values must have timestamps within the allowed ingestion window. Data older than the late data ingestion window is rejected. Future timestamps beyond a few minutes are rejected. Timestamps must be in epoch seconds with optional nanosecond offset. Duplicate timestamps for the same property overwrite previous values.

## Guardrail 9: Hierarchy Depth Has Limits
Asset hierarchies can be nested but have depth limits. Parent-child relationships are defined in the asset model. An asset can have multiple children but only one parent per hierarchy definition. Hierarchy changes require model updates. Deleting a parent asset requires removing children first.

## Guardrail 10: Property Data Types Must Match
Property values must match the data type defined in the asset model (STRING, INTEGER, DOUBLE, BOOLEAN). Type mismatches cause ingestion failures. The data type is set in the model and cannot be changed without creating a new property. Transforms and metrics inherit their data type from the formula result.

## Guardrail 11: Batch Operations Have Entry Limits
BatchPutAssetPropertyValue accepts up to 10 entries per call, each with up to 10 property values. BatchGetAssetPropertyValue and BatchGetAssetPropertyValueHistory have their own entry limits. Exceeding limits returns validation errors, not partial success.

## Guardrail 12: Alarms Follow a State Machine
SiteWise alarms have states: NORMAL, ACTIVE, LATCHED, SNOOZE_DISABLED, ACKNOWLEDGED. State transitions follow defined rules. Alarm evaluation depends on the metric or measurement value crossing a threshold. Alarm actions (SNS, Lambda) trigger on state transitions, not continuously.
