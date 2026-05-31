---
name: amplify-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Amplify Hosting problems
  by analyzing build failures, deployment issues, custom domains, backend
  environments, SSR configuration, access control, and CDN performance using
  structured runbooks. Activate when: build failures, build settings wrong,
  environment variables missing, deployment failures, branch configuration
  issues, PR preview problems, custom domain not working, SSL certificate
  errors, backend environment issues, API integration failures, SSR rendering
  errors, compute configuration problems, access control misconfigured, basic
  auth not working, CDN caching issues, rewrite rules not matching, or the
  user says something is wrong with Amplify without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Amplify, Route 53, ACM, CloudFront,
  IAM, and CloudWatch permissions. Some operations require Amplify console access.
---

# AWS Amplify Hosting Diagnostics

## When to use

Any AWS Amplify Hosting investigation where the console alone is insufficient — build failures, deployment issues, custom domain configuration, SSR problems, backend integration, access control, or CDN performance troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws amplify list-apps --query 'apps[*].{Id:appId,Name:name,Platform:platform,Repository:repository}'
version: "1.0.0"
last_updated: "2025-04-12"
aws amplify get-app --app-id <app-id>
aws amplify list-branches --app-id <app-id> --query 'branches[*].{Name:branchName,Stage:stage,Status:activeJobSummary.status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws amplify list-jobs --app-id <app-id> --branch-name <branch> --max-results 10
```

### Step 2 — Domain deep dive

```
aws amplify get-job --app-id <app-id> --branch-name <branch> --job-id <job-id>
aws amplify get-branch --app-id <app-id> --branch-name <branch>
aws amplify list-domain-associations --app-id <app-id>
aws amplify get-domain-association --app-id <app-id> --domain-name <domain>
```

### Step 3 — Detailed investigation

```
aws amplify list-backend-environments --app-id <app-id>
aws amplify get-backend-environment --app-id <app-id> --environment-name <env>
aws cloudfront list-distributions --query 'DistributionList.Items[?Comment==`<app-id>`]'
aws logs describe-log-groups --log-group-name-prefix /aws/amplify/<app-id>
```

Read `references/amplify-guardrails.md` before concluding on any Amplify issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-app` | Check app configuration, platform, repository |
| `list-branches` | Check branch configuration and deployment status |
| `get-job` / `list-jobs` | Check build and deployment job details |
| `get-branch` | Check branch-specific settings and environment variables |
| `list-domain-associations` | Check custom domain configuration |
| `get-domain-association` | Check domain DNS and SSL status |
| `list-backend-environments` | Check backend environment configuration |
| CloudWatch Logs | Check build logs and SSR runtime logs |
| CloudFront | Check CDN distribution settings |

## Gotchas: AWS Amplify Hosting

- Build settings can come from amplify.yml OR console. The amplify.yml file in the repository root takes precedence over console build settings. If both exist, the file wins. This is a common source of confusion when console changes don't take effect.
- Environment variables have scope. App-level variables apply to all branches. Branch-level variables override app-level. Some variables are reserved (e.g., _LIVE_UPDATES). Secrets are not exposed in build logs.
- Custom domains require DNS verification. Amplify creates CNAME records for verification. DNS propagation can take up to 48 hours. SSL certificates are automatically provisioned via ACM. Subdomains and root domains have different configuration requirements.
- SSR requires Amplify Hosting compute. SSR apps use Lambda@Edge or CloudFront Functions. Compute settings affect cold start and memory. Not all frameworks are supported for SSR. Build output must include server-side artifacts.
- PR previews create temporary deployments. They use a subdomain pattern. They are automatically deleted when the PR is closed. They share the app's environment variables unless overridden. They can increase build minutes usage.
- Rewrite and redirect rules have order precedence. Rules are evaluated top to bottom. The first matching rule wins. Trailing slashes matter. Regex patterns are supported but must be valid.
- Branch auto-detection connects to the repository. It requires repository access permissions. New branches are automatically deployed. Pattern matching controls which branches are deployed. Disconnecting the repository stops auto-detection.
- Basic auth protects the entire branch. It cannot be applied to specific paths. Credentials are branch-specific. It uses HTTP Basic Authentication. It is not a substitute for application-level auth.
- Amplify uses CloudFront for CDN. Cache invalidation happens automatically on deploy. Custom cache headers can be set in custom headers configuration. CloudFront distribution settings are managed by Amplify.

## Anti-hallucination rules

1. Always cite specific app IDs, job IDs, or API responses as evidence.
2. amplify.yml takes precedence over console build settings. Never assume console settings apply when the file exists.
3. Custom domains require DNS verification. Never assume instant domain activation.
4. SSR requires compute configuration. Never assume static hosting supports SSR.
5. Environment variables have scope precedence. Never assume app-level vars override branch-level.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Build | A1-A3 | Build failures, build settings, environment variables |
| B — Deploy | B1-B3 | Deployment failures, branch config, PR previews |
| C — Domain | C1-C2 | Custom domain, SSL certificate |
| D — Backend | D1-D2 | Backend environment, API integration |
| E — SSR | E1-E2 | Server-side rendering, compute config |
| F — Auth | F1-F2 | Access control, basic auth |
| G — Performance | G1-G2 | CDN caching, rewrite rules |
| Z — Catch-All | Z1 | General troubleshooting |
