# AWS Amplify Hosting Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Amplify Hosting problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Amplify Hosting when the console alone isn't enough — build failures, deployment issues, custom domain configuration, SSL certificates, backend environments, API integration, server-side rendering, compute configuration, access control, basic authentication, CDN caching, and rewrite/redirect rules.

### Activate When

- Build failures or build timeout
- Build settings not applied correctly
- Environment variables missing or incorrect
- Deployment failures or stuck deployments
- Branch configuration issues
- PR preview not working
- Custom domain not resolving
- SSL certificate provisioning failures
- Backend environment errors
- API integration not working
- SSR rendering errors or timeouts
- Compute configuration issues
- Access control misconfigured
- Basic auth not protecting branch
- CDN caching stale content
- Rewrite/redirect rules not matching
- General Amplify errors without clear symptoms

---

## Skill Structure

```
amplify-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-build-failures.md
    ├── A2-build-settings.md
    ├── A3-environment-variables.md
    ├── B1-deployment-failures.md
    ├── B2-branch-config.md
    ├── B3-pr-previews.md
    ├── C1-custom-domain.md
    ├── C2-ssl-certificate.md
    ├── D1-backend-environment.md
    ├── D2-api-integration.md
    ├── E1-ssr-rendering.md
    ├── E2-compute-config.md
    ├── F1-access-control.md
    ├── F2-basic-auth.md
    ├── G1-cdn-caching.md
    ├── G2-rewrite-rules.md
    ├── Z1-general-troubleshooting.md
    ├── amplify-guardrails.md
    └── amplify-hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Build** | A1–A3 | Build failures, build settings, environment variables |
| **B — Deploy** | B1–B3 | Deployment failures, branch config, PR previews |
| **C — Domain** | C1–C2 | Custom domain, SSL certificate |
| **D — Backend** | D1–D2 | Backend environment, API integration |
| **E — SSR** | E1–E2 | Server-side rendering, compute config |
| **F — Auth** | F1–F2 | Access control, basic auth |
| **G — Performance** | G1–G2 | CDN caching, rewrite rules |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/amplify-guardrails.md` covering amplify.yml precedence over console settings, environment variable scope and precedence, custom domain DNS verification requirements, SSL certificate auto-provisioning behavior, SSR compute requirements, PR preview lifecycle, rewrite rule evaluation order, basic auth scope limitations, CloudFront CDN integration, build image and dependency caching, branch auto-detection behavior, and webhook vs repository polling.

---

## Investigation Workflow

1. **Triage** — Check app configuration, recent build/deploy jobs, branch status
2. **Deep Dive** — Examine build logs, domain associations, backend environments
3. **Detailed** — Review CloudWatch logs, CloudFront distribution, IAM permissions

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `amplify:Get*`, `amplify:List*`, `cloudfront:ListDistributions`, `logs:DescribeLogGroups`, `logs:GetLogEvents`, `route53:ListHostedZones`
- Access to the Amplify console for build log details
- Repository access for amplify.yml verification

---

## Usage Examples

```
# List all Amplify apps
aws amplify list-apps

# Check recent builds for a branch
aws amplify list-jobs --app-id d1234567890 --branch-name main --max-results 5

# Get build job details
aws amplify get-job --app-id d1234567890 --branch-name main --job-id 1

# Check custom domain status
aws amplify list-domain-associations --app-id d1234567890

# Check branch configuration
aws amplify get-branch --app-id d1234567890 --branch-name main
```

---

## License

MIT-0
