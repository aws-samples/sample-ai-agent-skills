---
name: migrationorchestrator-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Migration Hub
  Orchestrator problems by analyzing workflow templates, migration
  workflows, step execution, plugins, SAP migration, SQL Server
  migration, and following structured runbooks.
  Activate when: workflow template issues, migration workflow failures,
  step execution errors, plugin problems, SAP migration issues, SQL
  Server migration errors, or the user says something is wrong with
  Migration Hub Orchestrator.
compatibility: >
  Requires AWS CLI or SDK access with migrationhuborchestrator,
  migration-hub-refactor-spaces, ec2, iam, cloudtrail, and ssm permissions.
---

# AWS Migration Hub Orchestrator Diagnostics

## When to use

Any Migration Hub Orchestrator investigation — workflow template selection, migration workflow creation, step execution failures, plugin configuration, SAP or SQL Server migration issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws migrationhuborchestrator list-templates --query 'templateSummary[*].{Id:id,Name:name}'
version: "1.0.0"
last_updated: "2025-04-12"
aws migrationhuborchestrator list-workflows --query 'migrationWorkflowSummary[*].{Id:id,Name:name,Status:status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws migrationhuborchestrator list-plugins --query 'plugins[*].{Id:pluginId,Name:hostname,Status:status}'
version: "1.0.0"
last_updated: "2025-04-12"
```

### Step 2 — Domain deep dive

```
aws migrationhuborchestrator get-workflow --id <workflow-id>
aws migrationhuborchestrator list-workflow-steps --workflow-id <workflow-id> --step-group-id <group-id>
aws migrationhuborchestrator get-workflow-step --workflow-id <workflow-id> --step-group-id <group-id> --id <step-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=migrationhuborchestrator.amazonaws.com --max-results 20
aws migrationhuborchestrator get-template --id <template-id>
aws migrationhuborchestrator list-workflow-step-groups --workflow-id <workflow-id>
```

Read `references/guardrails.md` before concluding on any Migration Hub Orchestrator issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `migrationhuborchestrator list-templates` | List workflow templates |
| `migrationhuborchestrator list-workflows` | List migration workflows |
| `migrationhuborchestrator get-workflow` | Get workflow details |
| `migrationhuborchestrator list-workflow-steps` | List steps in a workflow |
| `migrationhuborchestrator list-plugins` | List installed plugins |

## Gotchas: AWS Migration Hub Orchestrator

- Orchestrator uses predefined workflow templates for specific migration patterns (SAP, SQL Server, rehost, replatform). You cannot create custom templates from scratch.
- Plugins must be installed on source servers for certain workflow types. Plugin health must be verified before starting workflows.
- Workflow steps execute in order within step groups. Step groups can have dependencies. A failed step blocks subsequent steps.
- SAP migration workflows require specific SAP-aware steps and configurations. Generic migration templates don't handle SAP-specific requirements.
- SQL Server migration workflows integrate with DMS for data migration. DMS must be configured separately.
- Workflows can be paused and resumed but not restarted from the beginning. Failed steps can be retried individually.

## Anti-hallucination rules

1. Always cite specific workflow IDs, step statuses, or API responses as evidence.
2. Templates are predefined. Never suggest creating custom templates from scratch.
3. Plugins must be healthy before workflow execution. Never skip plugin validation.
4. Failed steps block subsequent steps. Never assume parallel execution.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Templates | A1-A2 | Template selection, template configuration |
| B — Workflows | B1-B2 | Workflow creation, workflow execution |
| C — Steps | C1-C2 | Step execution, step failures |
| D — Plugins | D1-D2 | Plugin installation, plugin health |
| E — SAP | E1-E2 | SAP migration, SAP-specific issues |
| F — SQL Server | F1 | SQL Server migration |
| Z — Catch-All | Z1 | General troubleshooting |
