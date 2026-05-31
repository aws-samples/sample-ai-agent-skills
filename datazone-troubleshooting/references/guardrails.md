# Amazon DataZone Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any DataZone issue.

## Guardrail 1: DataZone Is Governance and Catalog — Not Data Storage
DataZone manages metadata, governance policies, and data access. It does not store or process data. Actual data resides in Glue Data Catalog, Redshift, S3, or other sources. DataZone provides a governance layer on top.

## Guardrail 2: Domains Are the Top-Level Container
Everything in DataZone exists within a domain. Domains must be created first. Projects, assets, glossaries, and subscriptions are all domain-scoped. A domain has its own portal URL.

## Guardrail 3: Projects Organize Producers and Consumers
Projects group users who produce or consume data. Each project has its own environments for data access. Users must be project members to interact with project resources.

## Guardrail 4: Environments Connect to Actual Data Sources
DataZone environments are configured connections to data sources (Glue, Redshift, Athena). They provide the actual data access. Without environments, projects can catalog data but not access it.

## Guardrail 5: Subscriptions Require Approval
Data access between projects uses subscriptions. A consumer project requests a subscription to a producer's asset. The producer (or auto-approval policy) must approve before access is granted. This is by design for governance.

## Guardrail 6: Data Assets Are Metadata References
Data assets in the catalog are metadata entries that describe and reference data. They don't contain the actual data. Publishing an asset makes it discoverable; subscribing grants access to the underlying data source.

## Guardrail 7: Business Glossary Provides Standardized Terms
The business glossary defines standardized terminology. Glossary terms can be associated with data assets for consistent understanding. Glossary is domain-scoped.

## Guardrail 8: IAM and DataZone Permissions Are Separate
IAM controls who can call DataZone APIs. DataZone's internal permission model controls who can access projects, assets, and subscriptions within a domain. Both layers must be configured.

## Guardrail 9: Environment Blueprints Define Connection Types
Environment blueprints define what type of data source connection an environment provides (Glue, Redshift, etc.). Blueprints must be enabled in the domain before environments can use them.

## Guardrail 10: Data Quality Rules Execute in Underlying Services
Data quality rules defined in DataZone are executed by Glue Data Quality. DataZone provides the governance framework; Glue provides the execution engine.

## Guardrail 11: Domain Portal Is the User Interface
DataZone provides a web portal for each domain where users search the catalog, request subscriptions, and manage assets. The portal URL is domain-specific.

## Guardrail 12: Cross-Account Data Sharing Uses RAM
Cross-account data sharing in DataZone uses AWS RAM (Resource Access Manager) for sharing Glue databases and tables. RAM sharing must be configured for cross-account subscriptions.
