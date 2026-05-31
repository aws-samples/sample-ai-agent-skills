# AWS KMS Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any KMS issue.

## Guardrail 1: Key Policy Is the PRIMARY Access Control
Unlike most AWS services where IAM policies are the primary authorization mechanism, KMS key policies are the primary and mandatory access control. A KMS key policy MUST exist and MUST explicitly grant access. IAM policies for KMS only work if the key policy includes a statement allowing the account root principal (`arn:aws:iam::<account-id>:root`). Without this statement, IAM policies are completely ignored.

## Guardrail 2: Default Key Policy Enables IAM Policies
The default key policy created by AWS includes a statement granting the account root principal full KMS access. This is the statement that enables IAM policies to work for the key. If this statement is removed or modified, IAM policies lose their effect. Only principals explicitly named in the key policy can access the key. This is the most common cause of "IAM policy should work but doesn't" issues.

## Guardrail 3: Key Deletion Is PERMANENT After the Waiting Period
Scheduling key deletion puts the key in PendingDeletion state for 7-30 days (configurable, default 30). During this period, the key cannot be used but deletion CAN be cancelled. After the waiting period, the key is permanently deleted and ALL data encrypted with it becomes PERMANENTLY unrecoverable. There is no backup, no recovery, no AWS Support override after deletion completes.

## Guardrail 4: Disabled Keys Fail ALL Cryptographic Operations
A disabled key cannot perform any cryptographic operation — encrypt, decrypt, generate-data-key, sign, verify, or any other operation. AWS services using the key (S3, EBS, RDS, Lambda) will fail to read or write encrypted data. Re-enabling the key (`aws kms enable-key`) immediately restores all functionality. Disabling is reversible; deletion is not.

## Guardrail 5: Encryption Context Must Match on Decrypt
Encryption context is a set of non-secret key-value pairs cryptographically bound to the ciphertext during encryption. The EXACT same encryption context must be provided during decryption or the operation fails with InvalidCiphertextException. Encryption context is logged in CloudTrail (useful for auditing). It can be used in key policy and grant conditions for fine-grained access control.

## Guardrail 6: Grants Are for Temporary and Programmatic Access
Grants provide delegated, scoped permissions on a KMS key without modifying the key policy. They are commonly created by AWS services (EBS creates grants when attaching encrypted volumes, RDS creates grants for encrypted instances). Grants can specify constraints (encryption context). A grant can be retired by the grantee principal or revoked by the key administrator. Grants take effect immediately but may take up to 5 minutes to propagate (use grant tokens for immediate use).

## Guardrail 7: AWS-Managed Keys Cannot Be Modified by Customers
Keys with alias prefix `aws/` (aws/s3, aws/ebs, aws/rds, aws/lambda, etc.) are created and managed by AWS. Customers CANNOT: view or modify their key policies, manually rotate them, disable them, schedule them for deletion, create grants on them, or change any configuration. They rotate automatically every year. To use custom policies or rotation schedules, create a customer-managed key instead.

## Guardrail 8: Multi-Region Keys Are NOT Global Keys
Multi-region keys share the same key material and key ID across regions, but they are SEPARATE KMS keys in each region with separate ARNs, key policies, grants, and aliases. Creating a primary multi-region key does not automatically create replicas — you must explicitly replicate to each desired region. Each replica has its own key policy that must be configured independently.

## Guardrail 9: Request Rate Limits Vary by Key Type and Operation
KMS enforces per-account, per-region request rate limits. Cryptographic operations (Encrypt, Decrypt, GenerateDataKey, Sign, Verify) share a combined quota that varies by region (typically 5,500-30,000 requests/second for symmetric keys). Asymmetric operations have lower limits. Exceeding limits causes ThrottlingException. Use data key caching, envelope encryption, and request batching to reduce API calls.

## Guardrail 10: Key Material Origin Determines Key Capabilities
Keys can have three origins: AWS_KMS (AWS generates and manages key material), EXTERNAL (customer imports key material), or AWS_CLOUDHSM (key material in CloudHSM cluster). Imported key material does NOT support automatic rotation — you must manually rotate by importing new material. CloudHSM-backed keys require an active custom key store. The origin cannot be changed after key creation.

## Guardrail 11: Custom Key Stores Require Active CloudHSM Cluster
Custom key stores back KMS keys with an AWS CloudHSM cluster. The cluster must have at least two active HSMs in different AZs. If the custom key store is disconnected or the cluster is unavailable, ALL cryptographic operations on keys in that store fail. Custom key store keys have lower request rate limits than standard KMS keys.

## Guardrail 12: Key Spec and Usage Are Immutable After Creation
The key spec (SYMMETRIC_DEFAULT, RSA_2048, ECC_NIST_P256, etc.) and key usage (ENCRYPT_DECRYPT, SIGN_VERIFY, GENERATE_VERIFY_MAC) are set at key creation and CANNOT be changed. A key created for SIGN_VERIFY cannot be used for encryption. A symmetric key cannot be converted to asymmetric. To change the spec or usage, create a new key and re-encrypt data with it.
