---
name: lza-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Landing Zone Accelerator
  on AWS (LZA) problems by analyzing installation failures, pipeline execution,
  configuration files, CDK synthesis, CloudFormation stacks, upgrades, drift
  detection, and following structured runbooks.
  Activate when: LZA installation failures, pipeline errors, configuration file
  issues, CDK synth failures, CloudFormation stack errors, upgrade problems,
  drift detection, or the user says something is wrong with LZA.
compatibility: >
  Requires AWS CLI or SDK access with codepipeline, codebuild, cloudformation,
  organizations, s3, iam, sts, config, ec2, and cloudtrail permissions.
---

# Landing Zone Accelerator on AWS (LZA) Diagnostics

## When to use

Any Landing Zone Accelerator investigation — installation, pipeline execution, accounts/organization/network/security/global configuration, CDK synthesis, CloudFormation stack operations, upgrades, drift detection, or custom Config rules.

## Investigation workflow

### Step 1 — Collect and triage

```
aws codepipeline get-pipeline-state --name AWSAccelerator-Pipeline --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}'
aws cloudformation list-stacks --stack-status-filter CREATE_FAILED UPDATE_FAILED ROLLBACK_COMPLETE --query 'StackSummaries[?contains(StackName,`AWSAccelerator`)].{Name:StackName,Status:StackStatus}'
aws organizations list-accounts --query 'Accounts[*].{Id:Id,Name:Name,Status:Status}'
```

### Step 2 — Domain deep dive

```
aws codebuild batch-get-builds --ids <build-id> --query 'builds[0].{Status:buildStatus,Phase:currentPhase}'
aws cloudformation describe-stack-events --stack-name <stack-name> --max-items 20 --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`||ResourceStatus==`UPDATE_FAILED`]'
aws s3 ls s3://aws-accelerator-config-<account-id>-<region>/
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudformation.amazonaws.com --max-results 20
aws configservice describe-compliance-by-config-rule --query 'ComplianceByConfigRules[?Compliance.ComplianceType==`NON_COMPLIANT`]'
aws cloudformation detect-stack-drift --stack-name <stack-name>
```

Read `references/guardrails.md` before concluding on any LZA issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `codepipeline get-pipeline-state` | Check LZA pipeline execution status |
| `codebuild batch-get-builds` | Get CDK synth/deploy build details |
| `cloudformation describe-stack-events` | Investigate stack deployment failures |
| `cloudformation detect-stack-drift` | Check for configuration drift |
| `s3 ls` | Verify LZA configuration files |
| `organizations list-accounts` | Check account provisioning status |
| `configservice describe-compliance` | Check Config rule compliance |

## Gotchas: Landing Zone Accelerator

- LZA uses a single CodePipeline (AWSAccelerator-Pipeline) with multiple stages. Each stage deploys CloudFormation stacks to specific accounts and regions. Failures cascade — a failed stage blocks all subsequent stages.
- Configuration files (accounts-config.yaml, organization-config.yaml, network-config.yaml, security-config.yaml, global-config.yaml, customizations-config.yaml) are the primary input. Most LZA failures trace back to configuration file errors.
- LZA deploys CloudFormation stacks across multiple accounts and regions. A single configuration error can cause failures in dozens of stacks simultaneously.
- CDK synthesis (cdk synth) runs before deployment. Synth failures indicate configuration parsing errors or CDK code issues — not deployment problems.
- LZA manages AWS Organizations, SCPs, and account creation. Changes to these outside of LZA cause drift that can break subsequent pipeline runs.
- Upgrades must follow the documented version path. Skipping versions can cause state inconsistencies and deployment failures.
- Custom Config rules and customizations-config.yaml extend LZA but must follow strict schema requirements.

## Anti-hallucination rules

1. Always cite specific pipeline stages, stack names, or CloudFormation events as evidence.
2. Configuration file errors and deployment errors are different failure modes. Diagnose at the correct layer.
3. LZA manages resources across multiple accounts. Always identify which account and region is affected.
4. CDK synth failures are NOT CloudFormation failures. They occur before any deployment.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Installation/Pipeline | A1–A2 | Installation failures, pipeline execution errors |
| B — Account/Org Config | B1–B2 | Accounts config errors, organization config errors |
| C — Network/Security | C1–C2 | Network config failures, security config errors |
| D — Global/Custom Config | D1–D2 | Global config issues, customizations config |
| E — CDK/CloudFormation | E1–E2 | CDK synth failures, CloudFormation stack errors |
| F — Maintenance | F1–F2 | Upgrade failures, drift detection |
| G — Extensions | G1 | Custom Config rules |
| Z — Catch-All | Z1 | General troubleshooting |
