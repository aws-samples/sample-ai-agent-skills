# AWS IoT TwinMaker Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS IoT TwinMaker issue.

## Guardrail 1: Workspaces Require S3 Buckets
Every workspace needs a dedicated S3 bucket for content storage (scenes, 3D models, etc.). The bucket must exist before workspace creation. The workspace IAM role must have read/write access. The bucket should be in the same region as the workspace.

## Guardrail 2: Component Types Define Schema
Component types define the property schema (name, type, data connector). Components are instances of component types attached to entities. Changing a component type affects all entities using it. Component types are workspace-scoped.

## Guardrail 3: Data Connectors Use Lambda Functions
TwinMaker does not directly connect to databases. Data connectors invoke Lambda functions that handle external data retrieval. The Lambda function must return data in TwinMaker's expected format. The function needs IAM permissions for the data source.

## Guardrail 4: Scenes Are 3D Visualizations in S3
Scenes are JSON definitions stored in the workspace S3 bucket. They reference 3D models (glTF/GLB) also in S3. Scenes are not dashboards — they provide spatial 3D visualization. Scene rendering happens in the browser.

## Guardrail 5: Sync Jobs Are Not Real-Time
Sync jobs synchronize entity definitions from external sources periodically. They are not real-time. Sync jobs can fail due to conflicts, permissions, or source unavailability. Check sync job status for errors.

## Guardrail 6: Entity Hierarchy Has Depth Limits
Entities can have parent-child relationships forming a hierarchy. Hierarchy depth has limits. Circular references are rejected. Deleting a parent entity requires handling children first.

## Guardrail 7: Property Values Are Retrieved On-Demand
Properties with data connectors fetch values when requested, not continuously. Each request invokes the Lambda data connector. Caching is not built-in — implement caching in the Lambda function if needed.

## Guardrail 8: Workspace IAM Role Is Critical
The workspace IAM role must have permissions for S3 (content storage), Lambda (data connectors), IoT SiteWise (if using SiteWise connector), and any other integrated services. Missing permissions cause silent failures.

## Guardrail 9: 3D Models Must Be in Supported Formats
Scenes support glTF and GLB 3D model formats. Models must be uploaded to the workspace S3 bucket. Large models impact scene loading performance. Textures and materials must be embedded or referenced correctly.

## Guardrail 10: Grafana Integration Requires Plugin Configuration
The TwinMaker Grafana plugin must be installed and configured with workspace ID, region, and IAM authentication. Scene panels, property panels, and alarm panels each have specific configuration requirements.

## Guardrail 11: Pricing Is Based on API Calls and Storage
TwinMaker charges for API calls (GetPropertyValue, etc.), data connector invocations, and S3 storage. Frequent property value requests increase costs. Optimize query patterns and implement caching.

## Guardrail 12: Cross-Account Access Requires Explicit Configuration
Accessing TwinMaker workspaces from another account requires cross-account IAM role configuration. The workspace IAM role must trust the external account. Resource policies are not supported — use IAM roles.
