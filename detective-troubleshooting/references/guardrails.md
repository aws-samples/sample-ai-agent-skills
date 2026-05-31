# Detective Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Detective issue.

## Guardrail 1: GuardDuty Must Be Enabled Before Detective
Amazon Detective requires GuardDuty to be enabled and active for at least 48 hours before Detective can be enabled. GuardDuty findings are a primary data source for Detective. Without GuardDuty, Detective cannot create a behavior graph.

## Guardrail 2: Data Ingestion Takes Up to 24 Hours
After enabling Detective, adding a member account, or enabling a new data source, it takes up to 24 hours for data to appear in the behavior graph. Do not diagnose missing data issues until at least 24 hours have passed since enablement.

## Guardrail 3: Detective Is Regional — No Cross-Region Aggregation
Each AWS region has its own independent behavior graph. Detective does not aggregate data across regions. You must enable Detective in each region where investigation capabilities are needed and investigate findings in the region where they occurred.

## Guardrail 4: Behavior Graph Supports Maximum 1200 Member Accounts
A single behavior graph can have up to 1200 member accounts. For organizations with more accounts, you must use multiple behavior graphs or prioritize which accounts to include based on risk.

## Guardrail 5: Admin Account Cannot Be Changed Without Service Reset
The delegated administrator account for Detective (when using Organizations integration) cannot be changed without disabling Detective in the organization and re-enabling with a new admin account. This resets all behavior graph data.

## Guardrail 6: Member Accounts Must Accept Invitations
When not using Organizations auto-enable, member accounts must explicitly accept Detective invitations. Pending invitations do not contribute data to the behavior graph. Use Organizations integration for automatic enrollment.

## Guardrail 7: Historical Data Backfill Is Limited to 12 Months
When a new member account is added, Detective backfills up to 12 months of historical data from GuardDuty and other sources. Data older than 12 months is not available. The backfill process can take several hours.

## Guardrail 8: Automated Investigations Require Specific Data Sources
Automated investigation features require both GuardDuty and CloudTrail data source packages to be enabled. If either is missing, automated investigations produce incomplete results or fail to start.

## Guardrail 9: Detective Costs Are Based on Data Volume
Detective pricing is based on the volume of data ingested into the behavior graph. High-volume accounts (many CloudTrail events, VPC Flow Logs) can generate significant costs. Monitor usage during the 30-day free trial.

## Guardrail 10: VPC Flow Logs Data Source Is Optional
VPC Flow Logs ingestion is an optional data source package. It provides network-level visibility but increases data volume and costs. Enable only when network investigation capabilities are needed.

## Guardrail 11: EKS Audit Logs Require EKS Protection in GuardDuty
To ingest EKS audit logs into Detective, EKS Protection must be enabled in GuardDuty first. Detective cannot independently collect EKS audit logs without GuardDuty's EKS protection feature.

## Guardrail 12: Investigation Findings Are Not Exportable via API
While Detective provides investigation results in the console, the detailed visual investigation graphs and entity relationship maps are not fully exportable via API. The API provides indicators and metadata but not the full visual context.
