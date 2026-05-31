---
name: appconfig-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS AppConfig problems.
  Activate when: configuration deployment fails, feature flags not toggling,
  deployment strategy issues, configuration validation errors, environment
  or application creation failures, hosted configuration version problems,
  extension failures, or rollback issues.
compatibility: >
  Requires AWS CLI with appconfig permissions. For Lambda/ECS integration,
  requires AppConfig agent or Lambda extension.
---

# AWS AppConfig Diagnostics

## When to use

Any AppConfig investigation — configuration deployments, feature flags, validation errors, deployment strategies, extensions, or integration with Lambda/ECS/EC2.

## Investigation workflow

### Step 1 — Collect and triage

```
aws appconfig list-applications
aws appconfig list-environments --application-id <app-id>
aws appconfig list-configuration-profiles --application-id <app-id>
aws appconfig list-deployments --application-id <app-id> --environment-id <env-id>
aws appconfig get-deployment --application-id <app-id> --environment-id <env-id> --deployment-number <num>
```

### Step 2 — Deep dive

```
aws appconfig list-deployment-strategies
aws appconfig list-hosted-configuration-versions --application-id <app-id> --configuration-profile-id <profile-id>
aws appconfig list-extensions
aws appconfig list-extension-associations --resource-identifier <arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appconfig.amazonaws.com
```

### Step 3 — Detailed (low-confidence)

```
aws cloudwatch get-metric-statistics --namespace AWS/AppConfig --metric-name DeploymentComplete ...
aws logs filter-log-events --log-group-name <group> --filter-pattern "AppConfig"
```

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-applications` | List all AppConfig applications |
| `list-environments` | List environments for an application |
| `list-configuration-profiles` | List config profiles |
| `get-deployment` | Get deployment status and details |
| `list-deployment-strategies` | View deployment strategies |
| `get-hosted-configuration-version` | Retrieve configuration content |
| `list-extensions` | List AppConfig extensions |

## Gotchas

- Deployment strategies control rollout speed — Linear, Exponential, AllAtOnce have very different blast radii
- Configuration validators (JSON Schema or Lambda) run BEFORE deployment starts — failures block deployment
- AppConfig agent caches configurations locally — cache TTL affects how fast changes propagate
- Feature flags are a specific configuration type with built-in schema validation
- Hosted configuration has a 1MB size limit per version
- Extensions run at specific action points (PRE_CREATE_HOSTED_CONFIGURATION_VERSION, ON_DEPLOYMENT_START, etc.)
- Rollback triggers use CloudWatch alarms — alarm must be in ALARM state to trigger rollback

## Anti-hallucination rules

1. Always cite specific deployment status, error messages, or CloudTrail events as evidence
2. Deployment strategies are immutable after creation — you cannot modify an existing strategy
3. AppConfig is NOT the same as SSM Parameter Store — they are separate services with different APIs
4. Feature flags require the AWS.AppConfig.FeatureFlags configuration type
5. Never assume a deployment failed without checking the actual deployment state
6. Spend no more than 2 minutes on any single hypothesis

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Application/Environment | A1-A2 | Application creation, environment configuration |
| B — Configuration Profiles | B1-B2 | Profile creation, validation errors |
| C — Deployments | C1-C3 | Deployment failures, strategy issues, rollback |
| D — Feature Flags | D1-D2 | Flag configuration, toggle issues |
| E — Extensions | E1-E2 | Extension failures, association issues |
| F — Integration | F1-F2 | Lambda extension, ECS agent |
| Z — Catch-All | Z1 | General troubleshooting |
