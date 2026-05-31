---
title: "E2 — EC2 Integration"
description: "Diagnose CloudWatch Application Signals EC2 integration issues"
status: active
severity: MEDIUM
triggers: ["EC2 instrumentation", "CloudWatch agent EC2", "ADOT on EC2", "EC2 Application Signals"]
owner: devops-agent
objective: "Resolve EC2-specific Application Signals integration issues"
context: "EC2 integration requires manual installation of both CloudWatch agent and ADOT auto-instrumentation agent."
---

## Phase 1 — Triage

MUST:
- Verify CloudWatch agent is running: `aws ssm send-command --document-name AmazonCloudWatch-ManageAgent --parameters action=status --targets Key=InstanceIds,Values=<instance-id>`
- Check ADOT agent process is running on the instance
- Verify instance profile has CloudWatchAgentServerPolicy

SHOULD:
- Check CloudWatch agent configuration for application_signals section
- Verify ADOT agent environment variables (OTEL_SERVICE_NAME, OTEL_EXPORTER_OTLP_ENDPOINT)
- Check network connectivity to CloudWatch and X-Ray endpoints

MAY:
- Review CloudWatch agent log file on the instance
- Check ADOT agent log file for errors

## Phase 2 — Remediate

MUST:
- Install CloudWatch agent if missing: use SSM Distributor or manual installation
- Install ADOT auto-instrumentation agent for the application language
- Configure CloudWatch agent with application_signals section enabled

SHOULD:
- Set OTEL_SERVICE_NAME and OTEL_RESOURCE_ATTRIBUTES environment variables
- Configure ADOT agent to export to CloudWatch agent OTLP endpoint (localhost:4317)

MAY:
- Use SSM associations for automated agent management
- Create AMIs with pre-installed agents for new instances

## Common Issues

- symptoms: "CloudWatch agent running but no Application Signals metrics"
  diagnosis: "CloudWatch agent configuration does not include application_signals section."
  resolution: "Update CloudWatch agent configuration to include application_signals. Restart the agent."

- symptoms: "ADOT agent fails to start on EC2 instance"
  diagnosis: "Java agent JAR path is incorrect or JVM arguments are not set."
  resolution: "Verify -javaagent path points to the ADOT agent JAR. Ensure JAVA_TOOL_OPTIONS or JVM args are set correctly."

## Safety Ratings

```
safety_ratings:
  - "ssm send-command for agent status: GREEN — read-only status check"
  - "CloudWatch agent configuration: YELLOW — affects metric collection"
  - "ADOT agent installation: YELLOW — adds instrumentation to application"
  - "Application restart: YELLOW — brief service interruption"
```

## Escalation Conditions

- Agent installation fails due to OS compatibility issues
- Instrumentation causes significant application performance degradation on EC2
- Instance profile changes require IAM team involvement
- Large EC2 fleet requires coordinated agent deployment

## Data Sensitivity

- SSM command output: MEDIUM — may contain agent status and configuration
- CloudWatch agent configuration: MEDIUM — contains monitoring endpoints
- ADOT agent logs: MEDIUM — may contain application endpoint information
- Instance profile: MEDIUM — reveals IAM role configuration

## Prohibited Actions

- NEVER install agents on production EC2 instances without testing on staging
- NEVER modify application startup scripts without a rollback plan
- NEVER use root/administrator credentials for agent installation
- NEVER skip network connectivity verification before agent deployment

## Phase 3 — Rollback

1. Stop ADOT agent process on the instance
2. Remove ADOT agent JVM arguments from application startup
3. Restart the application without instrumentation
4. Revert CloudWatch agent configuration if modified
5. Verify application performance returns to baseline

## Output Format

```yaml
root_cause: "ec2_integration — <specific_cause>"
evidence:
  - type: agent_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix agent installation or configuration"
  long_term: "Automate agent deployment via SSM or AMI"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
