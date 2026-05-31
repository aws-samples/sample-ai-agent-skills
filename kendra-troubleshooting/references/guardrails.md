# Amazon Kendra Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Kendra issue.

## Guardrail 1: Index Creation Takes 15-30 Minutes
Creating a Kendra index provisions infrastructure including compute and storage. The index transitions through CREATING state before becoming ACTIVE. Queries against a non-ACTIVE index fail. Do not assume instant availability.

## Guardrail 2: Data Source Sync Is Asynchronous
After configuring a data source, you must trigger a sync via StartDataSourceSyncJob. Sync crawls, processes, and indexes documents. This can take minutes to hours depending on document volume. Documents are not searchable until sync completes.

## Guardrail 3: Query Relevance Depends on Document Quality
Kendra's relevance ranking uses ML models trained on document content and metadata. Poor results often stem from missing metadata, inconsistent formatting, or insufficient content. Custom relevance tuning requires experimentation with boosting and relevance feedback.

## Guardrail 4: FAQ Format Must Follow Specific Schema
FAQ files must be in CSV (question, answer columns) or JSON format with specific schema. Headers must match expected format. Encoding must be UTF-8. Format errors cause import failures with unhelpful error messages.

## Guardrail 5: Custom Document Enrichment Runs During Ingestion
Custom document enrichment (CDE) invokes Lambda functions during document ingestion. Lambda errors block document processing. CDE can modify metadata, filter documents, or transform content. Lambda timeout and error handling affect sync reliability.

## Guardrail 6: Access Control Uses Data Source ACLs
Kendra respects access control lists (ACLs) from data sources. If ACLs are configured, users only see documents they have access to. Misconfigured ACLs cause users to see no results. Token-based access control requires proper user context in queries.

## Guardrail 7: Experience Builder Requires IAM Identity Center
Kendra Experience Builder creates search applications with user authentication. It requires AWS IAM Identity Center (SSO) for user management. Without SSO configuration, experience creation fails.

## Guardrail 8: Index Editions Have Different Capabilities
Kendra Developer Edition has lower document limits and no high availability. Enterprise Edition supports more documents, higher query throughput, and HA. Edition cannot be changed after creation — you must create a new index.

## Guardrail 9: Data Source Connectors Have Specific Requirements
Each data source connector (S3, SharePoint, Salesforce, etc.) has specific configuration requirements including credentials, endpoint URLs, and crawl scope. Connector-specific errors require checking the connector documentation.

## Guardrail 10: Document Attributes Must Be Defined Before Use
Custom document attributes (metadata fields) must be defined in the index configuration before they can be used in queries or facets. Undefined attributes are silently ignored during ingestion.

## Guardrail 11: Query Suggestions Require Minimum Query History
Query suggestions are generated from query history. A minimum number of unique queries is required before suggestions appear. New indices have no suggestions until sufficient query volume accumulates.

## Guardrail 12: Thesauri and Synonyms Affect Search Behavior
Thesaurus files define synonym groups that expand search queries. Incorrect synonym definitions can degrade relevance by matching unrelated documents. Thesaurus upload is asynchronous and takes effect after processing.
