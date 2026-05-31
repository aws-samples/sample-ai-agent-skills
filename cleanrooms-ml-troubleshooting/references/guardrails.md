# AWS Clean Rooms ML Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Clean Rooms ML issue.

## Guardrail 1: Clean Rooms ML Requires an Active Collaboration
Clean Rooms ML features operate within a Clean Rooms collaboration. You must have an active collaboration with ML configured before training models or generating audiences. ML capabilities are not standalone — they depend on the collaboration framework.

## Guardrail 2: Training Datasets Have Strict Schema Requirements
Training datasets must conform to specific column schemas with exact data types. CSV files must match the declared schema precisely. Schema mismatches cause silent training failures or degraded model quality. Always validate schema before training.

## Guardrail 3: Privacy Budgets Are Finite and Non-Reversible
Privacy budget consumption is permanent per audience generation job. Once a privacy budget is exhausted, no more jobs can run until the budget period refreshes. Budget consumption cannot be reversed or refunded. Plan audience generation jobs carefully.

## Guardrail 4: Lookalike Models Require Minimum Seed Audience Size
Lookalike models require seed audiences meeting minimum size thresholds (typically 500+ records). Seed audiences below the minimum are rejected. The quality of lookalike output directly correlates with seed audience quality and size.

## Guardrail 5: Configured Models Must Be Associated with Collaborations
A configured audience model must be explicitly associated with a collaboration before it can be used for audience generation. The association defines output constraints, privacy parameters, and which collaboration members can trigger jobs.

## Guardrail 6: Differential Privacy Epsilon Is a Mathematical Tradeoff
Higher epsilon values reduce noise but weaken privacy guarantees. Lower epsilon values strengthen privacy but increase noise in outputs. There is no universally correct epsilon — it depends on the use case, data sensitivity, and regulatory requirements.

## Guardrail 7: S3 Data Must Be Co-Located with the Collaboration Region
Training data in S3 must reside in the same AWS region as the Clean Rooms collaboration. Cross-region S3 access is not supported. Verify bucket region matches collaboration region before troubleshooting data access failures.

## Guardrail 8: Audience Generation Jobs Are Asynchronous
Audience generation jobs run asynchronously and may take minutes to hours depending on data size and model complexity. Job status must be polled. Do not assume a job has failed simply because it has not completed quickly.

## Guardrail 9: KMS Encryption Is Required for Sensitive Data
Clean Rooms ML requires KMS encryption for data at rest. The KMS key policy must grant access to the Clean Rooms ML service principal. Missing KMS permissions cause cryptic access denied errors that may appear as S3 permission issues.

## Guardrail 10: Model Training Is Not Instantaneous
Model training can take hours depending on dataset size and complexity. Training status transitions through CREATING, ACTIVE, or CREATE_FAILED states. Do not interrupt training by deleting resources prematurely.

## Guardrail 11: Output Configuration Defines Where Results Land
Output configuration specifies the S3 destination for audience generation results. The destination bucket must have proper permissions for the Clean Rooms ML service role. Misconfigured output causes jobs to fail at the write stage after consuming privacy budget.

## Guardrail 12: IAM Roles Need Both Clean Rooms and Clean Rooms ML Permissions
Service roles for Clean Rooms ML need permissions for both cleanrooms and cleanroomsml API actions, plus S3 and KMS access. Missing permissions on either service cause failures. The trust policy must allow both service principals.
