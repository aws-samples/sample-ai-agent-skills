# AWS Glue DataBrew Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Glue DataBrew issue.

## Guardrail 1: DataBrew Is NOT Glue ETL
DataBrew and Glue ETL are different services under the Glue umbrella. DataBrew uses `databrew:` IAM actions, visual recipes, and its own API. Glue ETL uses `glue:` IAM actions, PySpark/Scala scripts, and different APIs. Do NOT suggest Glue ETL solutions for DataBrew problems.

## Guardrail 2: Recipe Jobs and Profile Jobs Are Different
Recipe jobs apply data transformations (cleaning, formatting). Profile jobs analyze data statistics and quality. They have different configurations, IAM requirements, and outputs. Do NOT confuse their settings or error patterns.

## Guardrail 3: DataBrew Requires an IAM Role with S3 Access
Every DataBrew job needs an IAM role. The role must have S3 read access to input data and S3 write access to output locations. The role also needs the `databrew.amazonaws.com` trust policy. Missing S3 permissions are the most common job failure cause.

## Guardrail 4: Datasets Reference Data, They Don't Store It
A DataBrew dataset is a pointer to data in S3, a JDBC connection, or a Glue Data Catalog table. The dataset configuration defines format, delimiter, and location. The actual data remains in the source. Dataset issues are usually connection or format problems.

## Guardrail 5: Data Quality Rulesets Are Separate from Recipes
Data quality rules validate data (null checks, range checks, pattern matching). They are defined as rulesets and attached to profile jobs. Recipes transform data. Do NOT confuse data quality validation with recipe transformations.

## Guardrail 6: Output Configuration Is Per Job, Not Per Project
Each job independently configures output format (CSV, JSON, Parquet, etc.), compression, S3 location, and overwrite behavior. Project settings don't control job output. Check the specific job configuration for output issues.

## Guardrail 7: DataBrew Does NOT Support Streaming
DataBrew processes batch data only. It reads from S3, databases, or Data Catalog. It does NOT support Kinesis streams, Kafka, or real-time data. For streaming, use Glue ETL streaming jobs or Kinesis Data Analytics.

## Guardrail 8: Recipe Steps Execute in Order
Recipe steps are applied sequentially. Step order matters — a filter step before a transformation produces different results than the reverse. When debugging recipe output, check step order and intermediate results.

## Guardrail 9: DataBrew Has Separate Quotas from Glue ETL
DataBrew has its own service quotas (concurrent jobs, datasets, projects). These are separate from Glue ETL quotas. Check DataBrew-specific quotas in Service Quotas, not Glue ETL quotas.

## Guardrail 10: Profile Job Statistics Depend on Sample Size
Profile jobs can run on a full dataset or a sample. Sample-based profiles may miss rare values or patterns. Statistics accuracy depends on sample size. For accurate profiling, use full dataset mode or increase sample size.

## Guardrail 11: Schedule Expressions Use Cron Syntax
DataBrew schedules use cron expressions (e.g., `cron(0 12 * * ? *)`). The cron format follows AWS EventBridge syntax with 6 fields. Invalid cron expressions cause schedule creation failures. Day-of-week and day-of-month cannot both be specified (use ? for one).

## Guardrail 12: DataBrew Job Logs Go to CloudWatch
Job execution logs are in CloudWatch Logs under `/aws-glue-databrew/`. Each job has its own log stream. Check CloudWatch Logs for detailed error messages, not just the job run status in the DataBrew console.
