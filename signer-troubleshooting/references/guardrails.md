# AWS Signer Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Signer issue.

## Guardrail 1: Signing Profiles Are Immutable
Once created, signing profiles cannot be modified. You cannot change the platform, signing parameters, or key configuration. To change settings, create a new profile. The old profile remains valid for existing signatures.

## Guardrail 2: Profile Revocation Is Irreversible
Revoking a signing profile invalidates ALL signatures made with that profile — past, present, and future. This can break deployed Lambda functions and IoT firmware. Always verify the blast radius before revoking.

## Guardrail 3: Lambda and IoT Use Different Platforms
Lambda code signing uses the `AWSLambda-SHA384-ECDSA` platform. IoT code signing uses `AmazonFreeRTOS-TI-CC3220SF` or `AWSIoTDeviceManagement-SHA256-ECDSA` platforms. Using the wrong platform causes signing failures.

## Guardrail 4: Signing Jobs Are Asynchronous
Signing jobs run asynchronously. StartSigningJob returns immediately with a job ID. The actual signing may take seconds to minutes depending on artifact size. Poll with DescribeSigningJob or use notifications.

## Guardrail 5: AWS Manages the Signing Keys
For most platforms, AWS Signer manages the signing keys via KMS. You cannot import or export signing keys. The key is created when the signing profile is created and is tied to the profile lifecycle.

## Guardrail 6: Code Signing Config Enforcement Modes
Lambda code signing configs have two modes: Warn (logs invalid signatures but allows deployment) and Enforce (blocks deployment of unsigned or invalidly signed code). Switching to Enforce can break CI/CD pipelines.

## Guardrail 7: S3 Source and Destination Required
Signing jobs require source artifacts in S3 and write signed artifacts to S3. Both the source and destination buckets must be accessible to the Signer service. Bucket policies and KMS encryption can block access.

## Guardrail 8: Signature Validity Period
Signing profiles have a signature validity period. After this period, signatures expire and are no longer valid. Lambda functions with expired signatures may fail validation if the code signing config enforces validity.

## Guardrail 9: Cross-Account Signing Requires Resource Policy
Cross-account signing requires a resource policy on the signing profile granting `signer:StartSigningJob` to the target account. Without this, cross-account signing jobs fail with AccessDeniedException.

## Guardrail 10: Profile Versioning
Signing profiles support versioning. Each version has its own signing parameters. When referencing a profile, specify the version if needed. The latest version is used by default.

## Guardrail 11: Notification Configuration Uses SNS
Signer notifications use SNS topics. The SNS topic must have a resource policy allowing Signer to publish. Notifications are sent for job status changes (Succeeded, Failed). Missing SNS permissions cause silent notification failures.

## Guardrail 12: Regional Service
AWS Signer is regional. Signing profiles, jobs, and configurations exist in the region where they are created. Cross-region signing requires profiles in each region. Lambda code signing configs must reference profiles in the same region.
