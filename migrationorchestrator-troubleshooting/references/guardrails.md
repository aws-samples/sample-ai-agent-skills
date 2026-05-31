# AWS Migration Hub Orchestrator Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Migration Hub Orchestrator issue.

## Guardrail 1: Templates Are Predefined
Workflow templates are AWS-provided for specific migration patterns. You cannot create custom templates from scratch. Choose the template that best matches your migration scenario and customize parameters within the template.

## Guardrail 2: Plugins Must Be Healthy
Certain workflow types require plugins installed on source servers. Plugin health must be verified before starting workflows. Unhealthy plugins cause step failures. Check plugin status: aws migrationhuborchestrator list-plugins.

## Guardrail 3: Steps Execute Sequentially Within Groups
Steps within a step group execute in order. A failed step blocks all subsequent steps in the group. Step groups can have dependencies on other groups. Understand the execution order before troubleshooting.

## Guardrail 4: Workflows Cannot Be Restarted
Workflows can be paused and resumed but cannot be restarted from the beginning. Failed steps can be retried individually. If a workflow is in a bad state, you may need to create a new workflow.

## Guardrail 5: SAP Workflows Require SAP-Specific Configuration
SAP migration workflows have SAP-specific steps (system copy, database migration, application server migration). These require SAP credentials, system IDs, and SAP-aware configuration. Generic migration templates don't handle SAP requirements.

## Guardrail 6: SQL Server Workflows Integrate with DMS
SQL Server migration workflows use AWS DMS for data migration. DMS replication instances, endpoints, and tasks must be configured separately. Orchestrator coordinates the workflow but DMS handles actual data migration.

## Guardrail 7: Manual Steps Require User Action
Some workflow steps are manual — they require user action (validation, approval, configuration) before the workflow can proceed. Manual steps pause the workflow until marked complete.

## Guardrail 8: Workflow Parameters Are Set at Creation
Most workflow parameters are set during creation and cannot be changed after the workflow starts. Verify all parameters before creating the workflow.

## Guardrail 9: Step Outputs Feed Into Subsequent Steps
Steps can produce outputs that are used as inputs for subsequent steps. If a step produces incorrect output, downstream steps may fail even if they're configured correctly.

## Guardrail 10: Orchestrator Is Regional
Migration Hub Orchestrator is regional. Workflows and templates are specific to the region. For cross-region migrations, the workflow runs in the target region.

## Guardrail 11: IAM Roles Must Cover All Step Actions
The workflow execution role must have permissions for all actions performed by all steps. Different steps may require different service permissions (EC2, DMS, SSM, etc.).

## Guardrail 12: Plugin Communication Uses SSM
Plugins on source servers communicate with Orchestrator through SSM. Source servers must have SSM agent installed and connectivity to SSM endpoints.
