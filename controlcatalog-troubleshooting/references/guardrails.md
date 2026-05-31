# AWS Control Catalog Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Control Catalog issue.

## Guardrail 1: Control Catalog Is Read-Only
Control Catalog is a read-only service that provides information about available controls. It does NOT enable, disable, or enforce controls. Use Control Tower, Config, or other services to implement controls.

## Guardrail 2: Controls Are Organized by Domains and Objectives
Controls are categorized into domains (broad categories like Security, Compliance) and objectives (specific goals within domains). Understanding this hierarchy is essential for finding relevant controls.

## Guardrail 3: Common Controls Are AWS-Defined
Common controls are AWS-recommended controls that map to multiple compliance frameworks. They are predefined by AWS and cannot be customized. They represent best practices applicable across frameworks.

## Guardrail 4: Not All Controls Are in the Catalog
Control Catalog is continuously expanding. Not all controls from all AWS services may be listed. The absence of a control in the catalog doesn't mean it doesn't exist — it may not yet be cataloged.

## Guardrail 5: Regional Availability Varies
Some controls are only available in specific AWS regions. Check control details for regional support before planning implementation. Not all controls are available in all regions.

## Guardrail 6: Catalog Controls vs Enabled Controls
Controls listed in the catalog are AVAILABLE controls. They are not necessarily ENABLED in your environment. Use Control Tower APIs to check which controls are enabled: aws controltower list-enabled-controls.

## Guardrail 7: Control ARNs Are Unique Identifiers
Each control has a unique ARN. Use ARNs for precise control identification. Control names may be similar across different domains or objectives.

## Guardrail 8: API Pagination Is Required
Control Catalog APIs return paginated results. Large result sets require multiple API calls with pagination tokens. Always handle pagination to get complete results.

## Guardrail 9: Control Catalog API Is Separate from Control Tower
Control Catalog (controlcatalog) and Control Tower (controltower) are separate APIs. Catalog provides information about available controls. Control Tower manages control enablement and enforcement.

## Guardrail 10: Filtering Uses Specific Syntax
Control listing supports filtering by various criteria. Filter syntax must match the API specification. Incorrect filters return empty results, not errors.

## Guardrail 11: Control Behavior Descriptions Are Informational
Control descriptions in the catalog explain what the control does but may not include all implementation details. Refer to service-specific documentation for implementation guidance.

## Guardrail 12: Control Catalog Requires IAM Permissions
Accessing Control Catalog requires controlcatalog:* IAM permissions. These are separate from Control Tower permissions. Ensure IAM policies include both if needed.
