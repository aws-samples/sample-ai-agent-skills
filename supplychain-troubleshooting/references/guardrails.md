# AWS Supply Chain Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Supply Chain issue.

## Guardrail 1: Instances Are the Top-Level Resource
All Supply Chain operations are scoped to an instance. Data integration flows, datasets, planning configurations, and connectors belong to a specific instance. Always verify the correct instance ID before troubleshooting.

## Guardrail 2: Data Integration Flows Can Fail Silently
Data integration flows may complete without errors but produce incomplete or incorrect data in the data lake. Always verify data quality in the destination dataset, not just flow execution status.

## Guardrail 3: Data Lake Schemas Are Predefined
The Supply Chain data lake uses predefined namespaces and schemas (e.g., asc, default). Custom data must conform to expected schemas. Schema mismatches cause silent data rejection or transformation errors.

## Guardrail 4: Planning Models Require Historical Data
Demand planning and supply planning models require sufficient historical data (weeks to months) to produce accurate forecasts. New instances with limited data will produce low-confidence or inaccurate results. Do not assume model errors when data is insufficient.

## Guardrail 5: Inventory Visibility Is Eventually Consistent
Inventory visibility aggregates data from multiple sources with different update frequencies. Discrepancies between sources may reflect timing differences, not data errors. Allow for propagation delays before diagnosing inconsistencies.

## Guardrail 6: Connectors Have Specific Prerequisites
Each connector type (SAP, EDI, custom) has specific configuration requirements including credentials, network access, and data format expectations. Connector failures often stem from prerequisite issues, not Supply Chain service problems.

## Guardrail 7: S3 Is the Primary Data Exchange
S3 buckets serve as the primary data exchange mechanism. File format (CSV, JSON, Parquet), folder structure, and naming conventions must match Supply Chain expectations. Incorrect file placement is a common cause of integration failures.

## Guardrail 8: Instance Creation Is Asynchronous
Instance creation can take several minutes. Do not assume creation has failed if the instance is in INITIALIZING state. Check back after allowing sufficient time for provisioning.

## Guardrail 9: Data Lake Datasets Have Namespace Scoping
Datasets are organized by namespace (e.g., asc for AWS Supply Chain managed, default for custom). Operations on datasets must specify the correct namespace. Cross-namespace references are not supported.

## Guardrail 10: Insights Require Data Completeness
Supply Chain insights and recommendations are generated based on data completeness and quality. Missing or incomplete data produces unreliable insights. Verify data integration completeness before questioning insight accuracy.

## Guardrail 11: Regional Availability Is Limited
AWS Supply Chain is available in limited regions. Verify the service is available in the target region before troubleshooting. Cross-region data access is not supported.

## Guardrail 12: BOM Import Has Specific Format Requirements
Bill of Materials (BOM) import jobs require specific CSV or JSON formats. Column names, data types, and relationships must match the expected schema. Format errors cause job failures with limited error details.
