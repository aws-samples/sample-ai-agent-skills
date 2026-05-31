---
name: codeartifact-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS CodeArtifact
  problems by analyzing repository configuration, authentication,
  package management, and following structured runbooks. Activate when:
  Repository creation failures, upstream connection issues, auth token
  problems, domain permission errors, npm/pip/Maven package issues,
  package publishing failures, version conflicts, domain configuration
  errors, cross-account access, CI/CD integration problems, build tool
  configuration issues, or the user says something is wrong with
  CodeArtifact without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with CodeArtifact, IAM, KMS,
  CloudTrail, and CloudWatch permissions.
---

# CodeArtifact Diagnostics

## When to use

Any CodeArtifact investigation where the console alone is insufficient — repository or domain configuration failures, authentication problems, package resolution issues, publishing errors, cross-account access, or CI/CD integration problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws codeartifact list-domains
aws codeartifact list-repositories
aws codeartifact describe-domain --domain <domain>
aws codeartifact describe-repository --domain <domain> --repository <repo>
aws codeartifact get-authorization-token --domain <domain> --query authorizationToken
```

### Step 2 — Deep dive

```
aws codeartifact list-repositories-in-domain --domain <domain>
aws codeartifact get-domain-permissions-policy --domain <domain>
aws codeartifact get-repository-permissions-policy --domain <domain> --repository <repo>
aws codeartifact list-packages --domain <domain> --repository <repo>
aws codeartifact get-repository-endpoint --domain <domain> --repository <repo> --format <npm|pypi|maven|nuget|swift|generic>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codeartifact.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any CodeArtifact issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `codeartifact list-domains` | List all domains |
| `codeartifact describe-domain` | Domain details, encryption, status |
| `codeartifact list-repositories` | List all repositories |
| `codeartifact describe-repository` | Repository details, upstreams |
| `codeartifact get-authorization-token` | Get auth token for package managers |
| `codeartifact get-domain-permissions-policy` | Domain resource policy |
| `codeartifact get-repository-permissions-policy` | Repository resource policy |
| `codeartifact list-packages` | List packages in a repository |
| `codeartifact describe-package-version` | Package version details |
| `codeartifact get-repository-endpoint` | Package manager endpoint URL |
| `codeartifact list-package-versions` | Version history for a package |
| `codeartifact login` | Configure package manager auth |

## Gotchas: CodeArtifact

- Auth tokens expire after 12 hours by default: Tokens from `get-authorization-token` default to 12 hours. Maximum is 12 hours. CI/CD pipelines must refresh tokens per job.
- Upstream resolution is ordered: Packages are resolved from upstream repositories in the order they are listed. The first match wins. Reorder upstreams to prioritize internal packages over public.
- Domain is the billing and access boundary: All repositories in a domain share the same KMS key and domain policy. Cross-domain access is not supported — use cross-account domain access instead.
- Package versions are immutable: Once published, a package version cannot be overwritten. You must publish a new version or delete and re-publish. This prevents supply chain attacks.
- npm requires scope or registry configuration: npm must be configured to use the CodeArtifact registry either globally or per-scope. Without configuration, npm defaults to the public registry.
- pip requires index-url configuration: pip must be configured with `--index-url` pointing to the CodeArtifact PyPI endpoint. The `--extra-index-url` option can cause dependency confusion attacks.
- Maven requires settings.xml configuration: Maven needs server credentials and repository configuration in settings.xml. The auth token must be refreshed before builds.
- External connections count toward limits: Each repository can have one external connection (e.g., npmjs, pypi). The external connection counts as an upstream repository.
- Cross-account requires domain policy: Cross-account access requires a domain permissions policy granting access to the remote account, plus IAM permissions in the remote account.

## Anti-hallucination rules

1. Always cite specific domain names, repository names, or API responses as evidence.
2. Auth tokens expire after 12 hours maximum — never claim longer validity.
3. Package versions are immutable — never suggest overwriting an existing version.
4. Cross-account requires domain policy — never suggest only IAM is sufficient.
5. External connections are limited to one per repository — never suggest multiple.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 18 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Repository | A1-A2 | Creation failures, upstream connections |
| B — Auth | B1-B2 | Auth token issues, domain permissions |
| C — Package | C1-C3 | npm issues, pip/PyPI issues, Maven issues |
| D — Publishing | D1-D2 | Package publishing, version conflicts |
| E — Domain | E1-E2 | Domain configuration, cross-account |
| F — Integration | F1-F2 | CI/CD integration, build tools |
| Z — Catch-All | Z1 | General CodeArtifact troubleshooting |
