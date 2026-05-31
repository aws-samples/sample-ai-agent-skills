# CodeArtifact Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any CodeArtifact issue.

## Guardrail 1: Auth Tokens Expire After 12 Hours Maximum
CodeArtifact authorization tokens from `get-authorization-token` have a maximum lifetime of 12 hours (configurable with `--duration-seconds`, max 43200). CI/CD pipelines must refresh tokens before each build job. Expired tokens cause authentication failures in all package managers.

## Guardrail 2: Package Versions Are Immutable
Once a package version is published to CodeArtifact, it cannot be overwritten. Attempting to publish the same version again fails. You must either publish a new version number, or delete the existing version first with `delete-package-versions` and re-publish.

## Guardrail 3: Upstream Resolution Order Matters
Packages are resolved from upstream repositories in the order they are configured. The first repository containing the requested package version wins. Incorrect ordering can cause internal packages to be shadowed by public packages (dependency confusion).

## Guardrail 4: One External Connection Per Repository
Each CodeArtifact repository can have at most one external connection to a public registry (npmjs, pypi, maven-central, nuget-org, etc.). To access multiple public registries, create separate repositories with different external connections and chain them as upstreams.

## Guardrail 5: Cross-Account Requires Domain Policy
Cross-account access to CodeArtifact requires a domain permissions policy granting access to the remote account. IAM permissions alone in the remote account are NOT sufficient. Both domain policy and IAM policy must be configured.

## Guardrail 6: Domain Is the Encryption and Billing Boundary
All repositories within a domain share the same KMS encryption key. You cannot use different KMS keys for different repositories in the same domain. Storage costs are billed at the domain level. Plan domain structure around encryption and billing requirements.

## Guardrail 7: pip extra-index-url Creates Dependency Confusion Risk
Using `--extra-index-url` with pip allows pip to check both CodeArtifact and PyPI, creating a dependency confusion attack vector. Always use `--index-url` (not `--extra-index-url`) to ensure pip only resolves from CodeArtifact. Use upstream connections for public package access.

## Guardrail 8: npm Scope Configuration Is Required
npm does not automatically use CodeArtifact. You must either configure the registry globally (`npm config set registry`) or per-scope (`npm config set @scope:registry`). Without this, npm resolves from the public npmjs registry.

## Guardrail 9: Maven settings.xml Must Include Server Credentials
Maven requires both a `<server>` entry with the auth token and a `<repository>` entry in settings.xml or pom.xml. Missing either causes authentication or resolution failures. The auth token in settings.xml must be refreshed before builds.

## Guardrail 10: Deleted Packages Can Be Re-Published
Unlike some registries, CodeArtifact allows re-publishing a version after it has been deleted. However, the deletion and re-publish are separate operations. Consumers may have cached the old version locally.

## Guardrail 11: Repository Endpoint URLs Are Format-Specific
Each package format (npm, pypi, maven, nuget, swift, generic) has a different endpoint URL. Using the wrong format endpoint causes resolution failures. Always use `get-repository-endpoint` with the correct `--format` parameter.

## Guardrail 12: Asset Size Limit Is 5 GB
Individual package assets cannot exceed 5 GB. This limit applies to each file within a package version. Large binary artifacts may need to be split or stored elsewhere (e.g., S3).
