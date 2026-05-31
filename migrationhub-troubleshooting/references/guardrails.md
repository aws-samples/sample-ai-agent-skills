# AWS Migration Hub Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Migration Hub issue.

## Guardrail 1: Home Region Must Be Set First
Migration Hub requires a home region before any operations. All migration tracking data is stored in the home region. The home region cannot be changed after it's set. Use `aws mgh describe-home-region` to verify.

## Guardrail 2: Migration Hub Tracks — It Doesn't Migrate
Migration Hub aggregates migration status from tools like DMS, SMS, CloudEndure, and Application Migration Service. It does not perform migrations itself. If a migration is failing, troubleshoot the specific migration tool, not Migration Hub.

## Guardrail 3: Strategy Recommendations Require Server Data
Strategy Recommendations needs server inventory data (CPU, memory, OS, dependencies) before generating recommendations. Data comes from Application Discovery Service agents, agentless collectors, or manual import. No data = no recommendations.

## Guardrail 4: Refactor Spaces Creates Real AWS Resources
Refactor Spaces environments create actual AWS resources: API Gateway, NLB, Lambda functions, and optionally Transit Gateway. These resources incur costs. Deleting a Refactor Spaces environment cleans up these resources.

## Guardrail 5: Application Grouping Is Logical Only
Grouping servers into applications in Migration Hub is a logical organization. It doesn't affect the actual migration process or create dependencies. It's for tracking and reporting purposes.

## Guardrail 6: Progress Update Streams Are Per-Tool
Each migration tool (DMS, SMS, etc.) creates its own progress update stream. Migration Hub aggregates these streams. If status isn't updating, check the specific tool's integration with Migration Hub.

## Guardrail 7: Discovery Data Has Retention Limits
Application Discovery Service data has retention periods. Exported data should be stored separately for long-term analysis. Data collection must be actively running to capture current state.

## Guardrail 8: IAM Roles Differ Per Migration Tool
Each migration tool that reports to Migration Hub needs its own IAM role with mgh:NotifyMigrationTaskState and related permissions. Missing permissions cause silent status update failures.

## Guardrail 9: Strategy Recommendations Has Its Own Data Collector
Migration Hub Strategy Recommendations has its own data collector (separate from ADS agents). It collects application-level data for strategy analysis. Both collectors may be needed for complete assessment.

## Guardrail 10: Refactor Spaces Uses Strangler Fig Pattern
Refactor Spaces implements the strangler fig pattern for incremental refactoring. Traffic is routed through an API Gateway that can split between legacy and new services. Routing rules control the traffic split.

## Guardrail 11: Cross-Account Migration Tracking Requires Delegation
For Organizations, Migration Hub can aggregate across accounts using delegated administrator. Each member account must authorize Migration Hub in the home region.

## Guardrail 12: Migration Hub Orchestrator Automates Workflows
Migration Hub Orchestrator provides pre-built migration workflow templates. It's separate from basic migration tracking. Orchestrator workflows have steps that must complete in order.
