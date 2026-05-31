# AWS Elastic Beanstalk Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Elastic Beanstalk problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Elastic Beanstalk when the console alone isn't enough — environment lifecycle issues, deployment failure analysis, health status debugging, configuration troubleshooting, scaling problems, Docker platform issues, worker tier debugging, and logging/monitoring integration.

### Activate When

- Environment creation failures
- Environment update failures or rollbacks
- Environment termination issues
- Platform version upgrade problems
- Deployment failures (all-at-once, rolling, immutable, blue/green)
- Deployment rollback issues
- Immutable vs rolling deployment confusion
- Red or yellow environment health
- Health check misconfiguration
- Enhanced health reporting issues
- .ebextensions configuration errors
- Environment variable problems
- Instance profile or IAM role issues
- Auto-scaling not triggering or scaling incorrectly
- Load balancer configuration problems
- Docker platform deployment failures
- Multi-container Docker issues
- Worker tier SQS polling failures
- Dead letter queue configuration
- Log retrieval failures
- CloudWatch log integration issues

---

## Skill Structure

```
elasticbeanstalk-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-update-failures.md
    ├── A3-termination-issues.md
    ├── A4-platform-version.md
    ├── B1-deployment-failures.md
    ├── B2-rollback.md
    ├── B3-immutable-vs-rolling.md
    ├── C1-red-yellow-health.md
    ├── C2-health-check-config.md
    ├── C3-enhanced-health.md
    ├── D1-ebextensions-errors.md
    ├── D2-environment-variables.md
    ├── D3-instance-profile.md
    ├── E1-auto-scaling-issues.md
    ├── E2-load-balancer-config.md
    ├── F1-docker-platform-issues.md
    ├── F2-multi-container.md
    ├── G1-worker-tier-sqs.md
    ├── G2-dead-letter-queue.md
    ├── H1-log-retrieval.md
    ├── H2-cloudwatch-integration.md
    ├── Z1-general-troubleshooting.md
    ├── elasticbeanstalk-guardrails.md
    └── elasticbeanstalk-hallucination-patterns.yaml
```

---

## Runbook Library (28 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Environment** | A1–A4 | Creation failures, update failures, termination issues, platform version |
| **B — Deployment** | B1–B3 | Deployment failures, rollback, immutable vs rolling |
| **C — Health** | C1–C3 | Red/yellow health, health check config, enhanced health |
| **D — Configuration** | D1–D3 | .ebextensions errors, environment variables, instance profile |
| **E — Scaling** | E1–E2 | Auto-scaling issues, load balancer config |
| **F — Docker** | F1–F2 | Docker platform issues, multi-container |
| **G — Worker** | G1–G2 | Worker tier SQS, dead letter queue |
| **H — Logging** | H1–H2 | Log retrieval, CloudWatch integration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/elasticbeanstalk-guardrails.md` covering .ebextensions execution order, environment rebuild vs update, platform branch vs version, immutable deployment ASG behavior, worker tier SQS architecture, enhanced health agent requirement, managed platform update risks, environment cloning limitations, Docker platform configuration, leader_only behavior, deployment policy selection, and environment tier immutability.

---

## Investigation Workflow

1. **Triage** — Describe environment status, check health, list recent error events, review CloudFormation stack events
2. **Deep Dive** — Examine environment resources, platform version, instance health, ASG and ELB configuration
3. **Detailed** — Validate configuration settings, check SQS queues, review IAM profiles, inspect CloudTrail and CloudWatch logs

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `elasticbeanstalk:*`, `ec2:Describe*`, `autoscaling:Describe*`, `elasticloadbalancing:Describe*`, `cloudformation:Describe*`, `s3:GetObject`, `sqs:GetQueueAttributes`, `iam:GetInstanceProfile`, `iam:GetRole`, `cloudwatch:GetMetricStatistics`, `logs:DescribeLogGroups`, `cloudtrail:LookupEvents`
- CloudWatch metrics enabled for Elastic Beanstalk
- CloudTrail logging enabled (recommended)

---

## Usage Examples

```
# Describe environment status and health
aws elasticbeanstalk describe-environments --environment-names my-env

# Get detailed health information
aws elasticbeanstalk describe-environment-health --environment-name my-env --attribute-names All

# List recent error events
aws elasticbeanstalk describe-events --environment-name my-env --severity ERROR --max-records 20

# Check per-instance health
aws elasticbeanstalk describe-instances-health --environment-name my-env

# Retrieve tail logs
aws elasticbeanstalk request-environment-info --environment-name my-env --info-type tail
aws elasticbeanstalk retrieve-environment-info --environment-name my-env --info-type tail

# Validate configuration before applying
aws elasticbeanstalk validate-configuration-settings --application-name my-app --environment-name my-env --option-settings file://options.json
```

---

## License

MIT-0
