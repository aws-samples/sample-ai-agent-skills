---
title: "E2 — Environment Configuration"
description: "Diagnose deployment environment configuration issues"
status: active
severity: MEDIUM
triggers: ["environment config error", "environment not found", "wrong environment", "environment setup failed"]
owner: devops-agent
objective: "Resolve CodeCatalyst deployment environment configuration issues"
context: "Environments are project-scoped deployment targets linked to AWS accounts. They are not AWS Elastic Beanstalk environments."
---

## Phase 1 — Triage

MUST:
- List project environments in CodeCatalyst console
- Verify environment is linked to the correct AWS connection
- Check workflow references the correct environment name

SHOULD:
- Verify the AWS connection associated with the environment is active
- Check environment-level configuration (variables, secrets)
- Review workflow YAML for environment references

MAY:
- Compare environment configuration across dev/staging/prod
- Check if environment was recently modified

## Phase 2 — Remediate

MUST:
- Link environment to the correct AWS connection
- Fix environment name references in workflow YAML
- Configure required environment variables and secrets

SHOULD:
- Set up environment approval gates for production
- Configure environment-specific deployment parameters
- Document environment configuration for team reference

MAY:
- Create separate environments for each deployment stage
- Implement environment promotion workflows

## Safety Ratings

```
safety_ratings:
  - "environment listing: GREEN — read-only enumeration"
  - "environment configuration: YELLOW — modifies deployment target settings"
  - "environment creation: YELLOW — creates new deployment target"
  - "environment deletion: RED — removes deployment target configuration"
```

## Escalation Conditions

- Environment configuration changes do not take effect
- Environment linked to wrong AWS account causing cross-account deployment
- Production environment configuration corrupted
- Environment approval gates blocking critical deployments

## Data Sensitivity

- Environment metadata: LOW — environment names and types
- Environment variables: MEDIUM — deployment configuration parameters
- Environment secrets: HIGH — encrypted credentials for deployment
- AWS connection details: MEDIUM — reveals target account information

## Prohibited Actions

- NEVER delete production environments without backup of configuration
- NEVER bypass environment approval gates
- NEVER store secrets as plain-text environment variables
- NEVER link test environments to production AWS accounts

## Phase 3 — Rollback

1. If environment configuration was modified, restore original settings
2. If environment was deleted, recreate with documented configuration
3. Verify workflows can deploy to the environment after changes
4. Confirm environment approval gates are properly configured
5. Document environment changes and notify the deployment team

## Common Issues

- symptoms: "Workflow fails with 'environment not found'"
  diagnosis: "Environment name in workflow YAML does not match configured environment."
  resolution: "Verify environment name in workflow matches exactly (case-sensitive)."

- symptoms: "Deployment goes to wrong AWS account"
  diagnosis: "Environment linked to incorrect AWS connection."
  resolution: "Update environment to use the correct AWS connection for the target account."

## Output Format

```yaml
root_cause: "environment_configuration — <specific_cause>"
evidence:
  - type: environment_details
    content: "<config data>"
severity: MEDIUM
mitigation:
  immediate: "Fix environment configuration"
  long_term: "Document environment setup and implement configuration validation"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
