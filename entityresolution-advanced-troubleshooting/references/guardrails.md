# AWS Entity Resolution Advanced Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any advanced Entity Resolution issue.

## Guardrail 1: Three Resolution Types with Different Behaviors
RULE_MATCHING uses deterministic rules you define. ML_MATCHING uses machine learning for probabilistic matching. PROVIDER uses third-party services (LiveRamp, TransUnion). Each has different configuration, accuracy, pricing, and troubleshooting approaches. Do not conflate them.

## Guardrail 2: Matching Workflows and ID Mapping Workflows Are Separate
Matching workflows resolve entities across datasets using rules or ML. ID mapping workflows map identifiers using provider services. They have different APIs, configurations, and use cases. Do not apply matching workflow troubleshooting to ID mapping issues.

## Guardrail 3: Schema Mappings Control Matching Quality
Schema mappings define how input data fields map to Entity Resolution attributes. Incorrect mappings cause records to be processed but not matched correctly — there is no error, just poor results. Always verify schema mappings when investigating matching quality issues.

## Guardrail 4: Provider Services Have Independent Error Modes
Provider services (LiveRamp, TransUnion, etc.) are external services with their own availability, rate limits, and error conditions. Provider errors may be due to subscription status, API limits, or service outages — not Entity Resolution configuration.

## Guardrail 5: Rule Order Matters in Rule-Based Matching
Rule-based matching evaluates rules sequentially in the defined order. More specific rules should come before broader rules. Rule order affects both matching accuracy and performance. Reordering rules changes matching results.

## Guardrail 6: ML Matching Requires Sufficient Data Quality and Volume
ML matching accuracy depends on data quality (completeness, consistency) and volume (minimum records). Small datasets, datasets with many null fields, or inconsistent formatting produce poor ML matching results. ML matching is not suitable for all use cases.

## Guardrail 7: Output Permissions Are Separate from Input Permissions
The Entity Resolution service role needs read permissions for input data (Glue tables, S3) AND write permissions for output locations (S3 buckets). Missing output permissions cause job failures after processing completes — data is processed but results cannot be written.

## Guardrail 8: Data Quality Issues Manifest as Poor Matching Not Errors
Poor data quality (inconsistent formatting, missing fields, duplicates) causes poor matching results but does NOT cause job failures or errors. The job completes successfully with low match rates. Always investigate data quality when match rates are unexpectedly low.

## Guardrail 9: Cross-Account Workflows Require Resource Policies
Cross-account Entity Resolution workflows require resource policies on Glue databases, S3 buckets, and KMS keys in the source accounts. The Entity Resolution service role must be able to assume roles or access resources across accounts.

## Guardrail 10: Privacy Configuration Affects Output Content
Privacy-enhancing configurations (hashing, encryption) affect what data appears in output. Enabling privacy features may cause output to look different from expected — hashed values instead of plain text. This is by design, not an error.

## Guardrail 11: Glue Table Schema Must Match Schema Mapping
The Glue table schema (column names and types) must match the schema mapping configuration. Schema mismatches cause job failures or incorrect field mapping. Always verify Glue table schema when creating or modifying schema mappings.

## Guardrail 12: Job Execution Has Processing Time Proportional to Data Size
Entity Resolution job execution time is proportional to the input data size and complexity. Large datasets with many potential matches take significantly longer. Do not assume a long-running job has failed — check job status before intervening.
