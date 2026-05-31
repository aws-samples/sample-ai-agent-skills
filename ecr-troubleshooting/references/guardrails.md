# ECR Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any ECR issue.

## Guardrail 1: Auth Tokens Expire After 12 Hours
ECR authorization tokens obtained via `get-login-password` or `get-authorization-token` are valid for exactly 12 hours. After expiry, all Docker push/pull operations fail with authentication errors. CI/CD pipelines must refresh tokens before each job or at regular intervals.

## Guardrail 2: Cross-Account Access Requires Both IAM and Repository Policy
To pull or push images cross-account, the calling principal needs IAM permissions in their account AND the ECR repository must have a resource policy granting access to that principal or account. Missing either one results in AccessDeniedException.

## Guardrail 3: Lifecycle Policies Evaluate Asynchronously
Lifecycle policy changes do not take effect immediately. ECR evaluates lifecycle rules periodically, typically within 24 hours. Do not expect images to be deleted immediately after creating or updating a lifecycle policy. Check `describe-images` after 24 hours to verify.

## Guardrail 4: Tag Immutability Prevents Overwriting Existing Tags
When tag immutability is enabled on a repository, pushing an image with a tag that already exists fails with ImageTagAlreadyExistsException. This is by design to prevent accidental overwrites. Use unique tags (e.g., git SHA) or disable immutability if mutable tags like `latest` are required.

## Guardrail 5: Replication Does NOT Copy Policies or Scanning Config
ECR replication copies only image data. Lifecycle policies, repository policies, scanning configurations, and tag immutability settings are NOT replicated. Each destination repository must be configured independently.

## Guardrail 6: Image Scanning Has Two Modes with Different Behaviors
Basic scanning (Clair-based) runs on push and produces a point-in-time report. Enhanced scanning (Inspector-based) provides continuous monitoring and re-scans when new CVEs are published. Do not confuse the two — basic scanning does NOT re-scan existing images.

## Guardrail 7: ECR Public Uses Separate Authentication and Endpoints
ECR Public (`public.ecr.aws`) uses `aws ecr-public get-login-password` and a different registry endpoint than private ECR. Private ECR commands (`aws ecr`) do not work with ECR Public repositories. The APIs are completely separate.

## Guardrail 8: Untagged Images Accumulate Without Lifecycle Policies
When you push a new image with an existing tag, the previous image becomes untagged. Without lifecycle policies to clean up untagged images, they accumulate indefinitely and increase storage costs. Always configure lifecycle policies for production repositories.

## Guardrail 9: Pull Rate Limits Apply Per Region Per Account
ECR enforces pull rate limits: authenticated pulls are limited to 1000/second per region. Unauthenticated pulls from ECR Public have stricter limits. Rate limit errors manifest as 429 Too Many Requests. Use VPC endpoints and regional caching to mitigate.

## Guardrail 10: KMS Encryption Requires Key Policy Access
When a repository uses KMS encryption, the pushing/pulling principal must have `kms:Decrypt` and `kms:GenerateDataKey` permissions on the KMS key. Cross-account access requires the KMS key policy to grant access to the remote account in addition to the repository policy.

## Guardrail 11: Image Manifest Size Limit is 4 MB
Individual image manifests cannot exceed 4 MB. This includes multi-architecture manifest lists. Exceeding this limit causes push failures. Large manifests typically indicate too many layers or excessive metadata.

## Guardrail 12: Pull-Through Cache Rules Require Upstream Credentials
Pull-through cache rules for Docker Hub, GitHub Container Registry, and other upstream registries require storing credentials in AWS Secrets Manager. Without valid credentials, cached pulls fail with authentication errors against the upstream registry.
