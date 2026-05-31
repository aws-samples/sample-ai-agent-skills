---
name: ecr-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Elastic Container
  Registry (ECR) problems by analyzing repository configuration, image
  management, authentication, and following structured runbooks. Activate when:
  Docker push/pull failures, authentication token errors, image scanning
  issues, lifecycle policy problems, tag immutability conflicts, repository
  creation failures, cross-account access denied, cross-region replication
  lag, repository policy errors, encryption configuration issues, pull
  throughput bottlenecks, caching problems, ECR Public gallery issues,
  or the user says something is wrong with ECR without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with ECR, ECR Public, IAM, KMS,
  CloudTrail, and CloudWatch permissions.
---

# ECR Diagnostics

## When to use

Any ECR investigation where the console alone is insufficient — push/pull failures, authentication problems, image scanning issues, lifecycle policy misconfigurations, replication failures, security policy errors, or performance bottlenecks.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ecr describe-repositories
aws ecr describe-images --repository-name <repo> --query 'imageDetails[*].{tag:imageTags,pushed:imagePushedAt,size:imageSizeInBytes,scan:imageScanStatus}'
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
aws ecr get-authorization-token
aws ecr describe-registry
```

### Step 2 — Deep dive

```
aws ecr get-repository-policy --repository-name <repo>
aws ecr get-lifecycle-policy --repository-name <repo>
aws ecr describe-image-scan-findings --repository-name <repo> --image-id imageTag=<tag>
aws ecr describe-image-replication-status --repository-name <repo> --image-id imageTag=<tag>
aws ecr describe-pull-through-cache-rules
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ecr.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/ECR --metric-name RepositoryPullCount --dimensions Name=RepositoryName,Value=<repo> --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

Read `references/guardrails.md` before concluding on any ECR issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ecr describe-repositories` | List repositories and their configurations |
| `ecr describe-images` | Image details, tags, scan status, push time |
| `ecr get-login-password` | Obtain Docker authentication token |
| `ecr get-authorization-token` | Get base64-encoded auth token details |
| `ecr get-repository-policy` | Check repository resource policy |
| `ecr get-lifecycle-policy` | View lifecycle policy rules |
| `ecr describe-image-scan-findings` | Vulnerability scan results |
| `ecr describe-image-replication-status` | Cross-region/account replication status |
| `ecr describe-pull-through-cache-rules` | Pull-through cache configuration |
| `ecr-public describe-repositories` | ECR Public repository details |
| `ecr batch-get-image` | Retrieve image manifests |
| `ecr list-tags-for-resource` | Repository tags |

## Gotchas: ECR

- Auth tokens expire after 12 hours: ECR authorization tokens are valid for exactly 12 hours. Docker clients must re-authenticate after expiry. CI/CD pipelines must refresh tokens before each build.
- Tag immutability is per-repository: Once enabled, you cannot push an image with a tag that already exists. This prevents overwriting latest or other mutable tags. Disable immutability or use unique tags.
- Lifecycle policies evaluate asynchronously: Policy changes don't take effect immediately. ECR evaluates lifecycle rules periodically (typically within 24 hours). Don't expect instant image cleanup.
- Image scanning is not real-time: Basic scanning uses the Clair engine and scans on push. Enhanced scanning uses Amazon Inspector and provides continuous scanning. Results may take minutes to appear.
- Cross-account pull requires both IAM and repository policy: The pulling account needs IAM permissions AND the repository must have a resource policy granting access to the pulling account's principal.
- ECR has a rate limit on pull operations: Authenticated pulls are limited to 1000 per second per region. Unauthenticated pulls from ECR Public are more restricted. Use VPC endpoints to avoid NAT gateway bottlenecks.
- Replication does not copy lifecycle policies: When replicating to another region or account, only images are replicated. Lifecycle policies, repository policies, and scanning configurations must be set up separately.
- ECR Public has separate authentication: ECR Public uses `aws ecr-public get-login-password` and authenticates to `public.ecr.aws`, not the private registry endpoint.
- Image manifest size limit is 4 MB: Individual image manifests cannot exceed 4 MB. Multi-architecture manifests (manifest lists) have the same limit.
- Untagged images accumulate: Pushing a new image with an existing tag makes the old image untagged. Without lifecycle policies, untagged images accumulate and increase storage costs.

## Anti-hallucination rules

1. Always cite specific repository names, image digests, or API responses as evidence.
2. Auth tokens expire after 12 hours — never claim they are permanent or long-lived.
3. Lifecycle policies are asynchronous — never claim immediate effect after policy creation.
4. Cross-account access requires BOTH IAM policy and repository policy — never suggest only one.
5. Replication does not copy policies — never claim lifecycle or repository policies replicate.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Push/Pull | A1-A3 | Push failures, pull failures, auth token issues |
| B — Images | B1-B3 | Image scanning, lifecycle policies, tag immutability |
| C — Repository | C1-C2 | Creation failures, cross-account access |
| D — Replication | D1-D2 | Cross-region replication, cross-account replication |
| E — Security | E1-E2 | Repository policy, encryption |
| F — Performance | F1-F2 | Pull throughput, caching |
| G — Public | G1-G2 | ECR Public, gallery issues |
| Z — Catch-All | Z1 | General ECR troubleshooting |
