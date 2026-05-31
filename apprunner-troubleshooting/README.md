# App Runner Diagnostics Skill

Agent skill for investigating and troubleshooting AWS App Runner problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for App Runner when the console alone isn't enough — deployment debugging, build failure analysis, runtime crash investigation, auto-scaling configuration, health check tuning, custom domain setup, VPC connector networking, source repository issues, security configuration, and observability setup.

### Activate When

- Deployment failures or stuck deployments
- Build failures for code-based sources
- Image pull errors from ECR
- Health check failures causing rollbacks
- Application crashes or restarts
- Port configuration mismatches
- Auto-scaling not working as expected
- Cold start latency issues
- Custom domain SSL certificate validation failures
- VPC connector connectivity problems
- Outbound connectivity failures to external services
- ECR source configuration issues
- GitHub source connection or build issues
- Instance role permission denied errors
- Secrets or environment variable not available
- CloudWatch logs not appearing
- X-Ray tracing not working
- General App Runner service issues

---

## Skill Structure

```
apprunner-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-deployment-failures.md
    ├── A2-build-failures.md
    ├── A3-image-pull-errors.md
    ├── B1-health-check-failures.md
    ├── B2-application-crashes.md
    ├── B3-port-configuration.md
    ├── C1-auto-scaling-issues.md
    ├── C2-cold-start.md
    ├── D1-custom-domain-ssl.md
    ├── D2-vpc-connector.md
    ├── D3-outbound-connectivity.md
    ├── E1-ecr-source-issues.md
    ├── E2-github-source-issues.md
    ├── F1-instance-role-permissions.md
    ├── F2-secrets-env-vars.md
    ├── G1-logging-issues.md
    ├── G2-xray-tracing.md
    ├── Z1-general-troubleshooting.md
    ├── apprunner-guardrails.md
    └── apprunner-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Deployment** | A1–A3 | Deployment failures, build failures, image pull errors |
| **B — Runtime** | B1–B3 | Health check failures, application crashes, port configuration |
| **C — Scaling** | C1–C2 | Auto-scaling issues, cold start latency |
| **D — Networking** | D1–D3 | Custom domain/SSL, VPC connector, outbound connectivity |
| **E — Source** | E1–E2 | ECR source issues, GitHub source issues |
| **F — Security** | F1–F2 | Instance role permissions, secrets/env vars |
| **G — Observability** | G1–G2 | Logging issues, X-Ray tracing |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/apprunner-guardrails.md` covering source type differences, auto-scaling behavior, health check configuration, custom domain certificates, VPC connector routing, environment variable handling, build vs runtime phases, deployment triggers, instance role trust policy, X-Ray daemon management, no container exec access, and PORT environment variable requirement.

---

## Investigation Workflow

1. **Triage** — List services, check service status, review recent operations and CloudWatch metrics
2. **Deep Dive** — Examine source configuration, health checks, auto-scaling, networking, custom domains
3. **Detailed** — Inspect instance configuration, VPC connectors, observability, CloudTrail events, ECR images

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `apprunner:*`, `ecr:DescribeImages`, `ecr:GetAuthorizationToken`, `iam:GetRole`, `iam:PassRole`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `acm:DescribeCertificate`, `route53:ListHostedZones`, `logs:GetLogEvents`
- CloudWatch Logs enabled for the App Runner service (enabled by default)
- App Runner service ARN or service name available

---

## Usage Examples

```
# List all App Runner services
aws apprunner list-services

# Get full service details
aws apprunner describe-service --service-arn arn:aws:apprunner:us-east-1:123456789012:service/my-service/abc123

# Check recent operations (deployments)
aws apprunner list-operations --service-arn arn:aws:apprunner:us-east-1:123456789012:service/my-service/abc123

# Check auto-scaling configuration
aws apprunner describe-auto-scaling-configuration \
  --auto-scaling-configuration-arn arn:aws:apprunner:us-east-1:123456789012:autoscalingconfiguration/my-config/1/abc123

# Check custom domain status
aws apprunner list-custom-domains --service-arn arn:aws:apprunner:us-east-1:123456789012:service/my-service/abc123
```

---

## License

MIT-0
