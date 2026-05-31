# AWS Resource Explorer Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Resource Explorer issue.

## Guardrail 1: One Aggregator Index Per Account
Only ONE aggregator index is allowed per AWS account. The aggregator collects resource data from all local indexes across regions. Attempting to create a second aggregator index fails. To move the aggregator, delete the existing one first and create a new one in the desired region.

## Guardrail 2: Local Indexes Required Per Region
Each region must have a local index for its resources to be discoverable. Without a local index in a region, resources in that region are invisible to Resource Explorer search, even with an aggregator index in another region.

## Guardrail 3: Search Results Are Not Real-Time
Resource Explorer search results may be delayed up to 36 hours after resource creation, modification, or deletion. Do not assume the absence of a resource in search results means it does not exist. Verify with the service-specific describe/list API.

## Guardrail 4: Views Filter Search Results
Views define what resources are visible in search results. Custom views with filters may hide resources. Always check the view configuration (included properties, filters) before concluding a resource is missing from the account.

## Guardrail 5: Not All Resource Types Are Supported
Resource Explorer supports a subset of AWS resource types. Unsupported resource types never appear in search results regardless of index configuration. Check the supported resource types list before investigating missing resources.

## Guardrail 6: Filter Expression Syntax Is Specific
Resource Explorer filter expressions use a specific query syntax with field names like resourcetype, region, tag, and id. Common errors include using wrong field names, missing colons, and incorrect quoting. The syntax is NOT the same as CloudWatch Logs Insights or Athena SQL.

## Guardrail 7: Index Creation Is Asynchronous
Index creation (both local and aggregator) is asynchronous. The index status transitions from CREATING to ACTIVE. Searches against a CREATING index return incomplete results. Wait for ACTIVE status before troubleshooting search issues.

## Guardrail 8: Cross-Region Search Requires Aggregator
Cross-region resource search requires an aggregator index. Searching from a region with only a local index returns only that region's resources. The aggregator must be ACTIVE and all source regions must have local indexes.

## Guardrail 9: Organization Integration Has Separate Setup
Organization-wide resource search requires additional setup beyond account-level indexes. Multi-account search uses AWS Organizations integration and requires delegated administrator configuration.

## Guardrail 10: Default View Shows All Indexed Resources
The default view created with the index shows all indexed resources without filters. If users report missing resources while using the default view, the issue is likely index coverage or resource type support, not view configuration.

## Guardrail 11: Index Deletion Removes All Indexed Data
Deleting an index removes all indexed resource data for that region. Re-creating the index requires re-indexing, which can take up to 36 hours. Do not delete indexes as a troubleshooting step without understanding the re-indexing delay.

## Guardrail 12: Search Permissions Are Separate from Resource Permissions
Having permission to search Resource Explorer (resource-explorer-2:Search) does not grant permission to access or modify the discovered resources. Search results show resource metadata only. Separate IAM permissions are needed to interact with discovered resources.
