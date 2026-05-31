# Amazon FinSpace Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any FinSpace issue.

## Guardrail 1: Managed Datasets and Managed kdb Are Different Features
FinSpace managed datasets provide a data catalog with views and permissions. Managed kdb Insights provides kdb+ database clusters for time-series analytics. They have different APIs and use cases.

## Guardrail 2: kdb Environments Require VPC Configuration
Managed kdb environments are deployed in a VPC with transit gateway connectivity. Network configuration must be set up before creating kdb clusters. Incorrect networking causes cluster creation failures.

## Guardrail 3: Changesets Are Immutable
Once a changeset is created and processed, it cannot be modified. To correct data, create a new changeset. Changesets form an append-only history of data changes.

## Guardrail 4: kdb Cluster Types Serve Different Purposes
HDB (Historical Database) clusters serve historical data. RDB (Real-time Database) clusters handle streaming data. Gateway clusters route queries. GP (General Purpose) clusters handle mixed workloads. Choose the right type for your use case.

## Guardrail 5: Superuser Has Full Environment Access
The superuser has unrestricted access to all resources in the environment. Superuser credentials should be secured, rotated regularly, and used only when necessary. Create regular users for day-to-day operations.

## Guardrail 6: Environment Creation Takes Time
FinSpace environment creation provisions infrastructure and can take 15-30 minutes. Do not assume failure if the environment is still in CREATING state within this window.

## Guardrail 7: kdb Cluster Scaling Is Not Automatic
kdb clusters have fixed node types and counts. They do not auto-scale. To change capacity, you must create a new cluster with different configuration. Plan capacity based on expected workload.

## Guardrail 8: Data Ingestion Uses S3 as Staging
Data is ingested into kdb databases via changesets that reference S3 objects. The data must be uploaded to S3 first, then a changeset is created referencing the S3 path.

## Guardrail 9: FinSpace Is Available in Limited Regions
FinSpace is not available in all AWS regions. Check region availability before creating environments. Managed kdb may have different region availability than managed datasets.

## Guardrail 10: Permission Groups Control Dataset Access
Managed dataset access is controlled through permission groups. Users must be added to permission groups to access datasets. Permission groups are environment-specific.

## Guardrail 11: kdb Clusters Need Initialization Scripts
kdb clusters can be configured with initialization scripts (q scripts) that run at startup. These scripts set up database schemas, load data, and configure the kdb+ environment.

## Guardrail 12: Environment Deletion Is Permanent
Deleting a FinSpace environment permanently removes all data, datasets, databases, and configurations. There is no undo. Export data before deletion.
