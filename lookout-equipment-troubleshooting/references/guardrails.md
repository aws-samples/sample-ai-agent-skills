# Amazon Lookout for Equipment Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Lookout for Equipment issue.

## Guardrail 1: Minimum Training Data Requirement
Lookout for Equipment requires a MINIMUM of 14 days of continuous sensor data for model training. AWS recommends 90+ days for optimal results. Training with insufficient data will fail or produce unreliable models. Always verify data duration before investigating training failures.

## Guardrail 2: Strict CSV Format Requirements
Sensor data must be in CSV format with specific structure: one CSV file per component, each in a named subfolder under the dataset S3 prefix. Timestamps must be the first column in ISO 8601 format. Column headers must match the schema definition exactly.

## Guardrail 3: Timestamps Must Be Monotonically Increasing
All timestamps in sensor data must be strictly monotonically increasing with no gaps, duplicates, or out-of-order entries. The sampling rate must be consistent within each component file. Violations cause ingestion failures that may not have clear error messages.

## Guardrail 4: Inference Schedulers Run on Fixed Intervals
Inference schedulers operate on fixed cadences: 5, 10, 15, 30 minutes, or 1 hour. You cannot trigger on-demand inference through the scheduler. The scheduler expects new data in S3 at each interval. Missing data at the scheduled time causes execution failures.

## Guardrail 5: Model Training Is Long-Running and Non-Interruptible
Model training can take hours to days depending on dataset size and complexity. There is no way to check intermediate progress, speed up training, or pause/resume. A training job in TRAINING status is normal and should not be treated as stuck unless it exceeds 72 hours.

## Guardrail 6: S3 Bucket Must Be in Same Region as Lookout for Equipment
The S3 bucket containing sensor data must be in the same AWS region as the Lookout for Equipment resources. Cross-region S3 access is not supported. Always verify region alignment before investigating access errors.

## Guardrail 7: IAM Role Requires Specific S3 and Lookout Permissions
The service role needs s3:GetObject, s3:GetBucketLocation, s3:ListBucket on the data bucket, plus lookoutequipment permissions. The trust policy must allow lookoutequipment.amazonaws.com to assume the role. Missing trust policy is a common but non-obvious failure.

## Guardrail 8: Labels Are Optional But Improve Model Quality
Labels (marking known anomaly periods) are optional for training. Without labels, the model uses unsupervised learning. With labels, it uses semi-supervised learning for better accuracy. Labels must cover the same time range as training data.

## Guardrail 9: Retraining Requires Original Data Accessibility
Retraining schedulers need access to the original training data plus new data in S3. If the original training data is moved, deleted, or the S3 bucket policy changes, retraining will fail. Always verify data persistence before setting up retraining.

## Guardrail 10: Limited Regional Availability
Amazon Lookout for Equipment is available in limited AWS regions (us-east-1, eu-west-1, ap-northeast-2, and select others). Before troubleshooting, verify the service is available in the target region. Attempting to use the service in unsupported regions produces confusing endpoint errors.

## Guardrail 11: Inference Input Must Match Training Schema
Inference input data must contain exactly the same components and sensor columns as the training data. Missing columns, extra columns, or renamed columns cause inference failures. Always compare inference schema against the original dataset schema.

## Guardrail 12: Data Ingestion Jobs Are Asynchronous
Data ingestion is asynchronous — the start-data-ingestion-job API returns immediately with a job ID. The actual ingestion runs in the background and can take minutes to hours. Always check job status with describe-data-ingestion-job before assuming success or failure.
