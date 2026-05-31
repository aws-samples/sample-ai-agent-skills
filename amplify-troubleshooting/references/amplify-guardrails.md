# AWS Amplify Hosting Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Amplify Hosting issue.

## Guardrail 1: amplify.yml Takes Precedence Over Console Build Settings
If an amplify.yml file exists in the repository root, it overrides ALL build settings configured in the Amplify console. Changes made in the console will NOT take effect when the file exists. To use console settings, remove or rename the amplify.yml file. The file supports per-branch build settings using branch-specific configurations.

## Guardrail 2: Environment Variables Have Scope Precedence
Branch-level environment variables override app-level variables with the same name. Variables set in amplify.yml override both. Some variable names are reserved by Amplify (e.g., _LIVE_UPDATES, AWS_*). Secrets are masked in build logs but available during build. Environment variables are NOT available at runtime for static sites — they must be injected during build.

## Guardrail 3: Custom Domains Require DNS Verification Before Activation
Amplify creates CNAME records that must be added to your DNS provider for verification. DNS propagation can take up to 48 hours. The domain status goes through: PENDING_VERIFICATION → PENDING_DEPLOYMENT → AVAILABLE. Root domains require additional ANAME/ALIAS records. Subdomains use standard CNAME records. Do not assume instant domain activation.

## Guardrail 4: SSL Certificates Are Auto-Provisioned via ACM
Amplify automatically provisions SSL certificates through AWS Certificate Manager. Certificate provisioning requires successful DNS verification. Certificates are managed by Amplify — you cannot use custom certificates. Certificate renewal is automatic. If DNS verification fails, SSL will not be provisioned. Check domain association status for certificate issues.

## Guardrail 5: SSR Requires Amplify Hosting Compute Platform
Server-side rendering (SSR) apps require the "WEB_COMPUTE" platform, not "WEB" (static). SSR uses Lambda@Edge or CloudFront Functions for server-side logic. Not all frameworks support SSR on Amplify. Build output must include server-side artifacts. Compute settings (memory, timeout) affect SSR performance. Static sites do NOT need compute.

## Guardrail 6: PR Previews Are Temporary Branch Deployments
PR previews create a new branch deployment with a unique subdomain. They are automatically deleted when the PR is closed or merged. They share app-level environment variables but can have overrides. PR previews consume build minutes. They require repository webhook integration to trigger automatically.

## Guardrail 7: Rewrite and Redirect Rules Are Evaluated Top to Bottom
Rules are evaluated in order — the first matching rule wins. A catch-all rule (e.g., /<*>) at the top will match everything and prevent subsequent rules from executing. Trailing slashes matter in path matching. Regex patterns must be valid JavaScript regex. 200 rewrites serve content without URL change; 301/302 redirects change the URL.

## Guardrail 8: Basic Auth Protects the Entire Branch
Basic authentication applies to ALL paths on a branch — it cannot be scoped to specific routes. Credentials are configured per branch. It uses standard HTTP Basic Authentication headers. It is NOT a substitute for application-level authentication. Basic auth is typically used for staging/preview environments, not production.

## Guardrail 9: Amplify Uses CloudFront for Content Delivery
Amplify automatically creates and manages a CloudFront distribution. Cache invalidation happens automatically after each deployment. Custom cache-control headers can be set in the customHeaders section of amplify.yml. You cannot directly modify the CloudFront distribution settings. CloudFront access logs are not enabled by default.

## Guardrail 10: Build Image and Dependency Caching Affect Build Speed
Amplify provides default build images with common runtimes. Custom build images can be specified. Dependency caching (node_modules, etc.) is enabled by default and speeds up builds. Cache can become stale — clearing cache may fix build issues. Build timeout is 30 minutes by default. Large dependencies may require increased timeout.

## Guardrail 11: Branch Auto-Detection Requires Repository Connection
Auto-detection monitors the connected repository for new branches. It requires proper webhook configuration. Pattern matching (include/exclude) controls which branches trigger deployments. Disconnecting the repository stops all auto-detection. Reconnecting requires re-authorization. Manual deployments are always available regardless of auto-detection.

## Guardrail 12: Webhooks vs Repository Polling for Build Triggers
Amplify uses webhooks (not polling) to detect repository changes. Webhook failures cause missed builds. Webhooks must be properly configured in the repository provider (GitHub, GitLab, Bitbucket, CodeCommit). Manual builds can be triggered via console or CLI regardless of webhook status. Check repository webhook settings if builds are not triggering.
