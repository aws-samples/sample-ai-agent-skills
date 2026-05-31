# AWS IoT FleetWise Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any IoT FleetWise issue.

## Guardrail 1: One Signal Catalog Per Account Per Region
Each AWS account can have only ONE signal catalog per region. All vehicle models reference signals from this single catalog. Do not attempt to create multiple signal catalogs. If a catalog exists, add signals to it rather than creating a new one.

## Guardrail 2: Manifest Lifecycle States Must Be ACTIVE
Model manifests and decoder manifests have lifecycle states: DRAFT and ACTIVE. Manifests in DRAFT state cannot be used to create vehicles or campaigns. You must explicitly activate manifests using update-model-manifest or update-decoder-manifest with status ACTIVE.

## Guardrail 3: Campaigns Target Fleets Not Individual Vehicles
Campaigns are associated with fleets, not individual vehicles. To collect data from a specific vehicle, the vehicle must be a member of the fleet targeted by the campaign. There is no way to create a campaign targeting a single vehicle directly.

## Guardrail 4: Decoder Manifests Must Match Model Manifests
Each decoder manifest is associated with a specific model manifest. The decoder manifest must define decoders for all signals in the model manifest. Mismatches between decoder and model manifests cause vehicle creation or campaign failures.

## Guardrail 5: Edge Agent Configuration Is Vehicle-Specific
The FleetWise Edge Agent (FWE) running on the vehicle requires configuration specific to that vehicle, including the vehicle name, endpoint, and credentials. A misconfigured edge agent will fail to connect or report data. Configuration is not automatically pushed from the cloud.

## Guardrail 6: S3 and Timestream Destinations Have Different Characteristics
S3 destinations batch data and are suitable for analytics workloads. Timestream destinations provide near-real-time data access. They have different IAM requirements, data formats, and cost models. Do not assume they are interchangeable.

## Guardrail 7: Signal Catalog Changes Affect All Dependent Resources
Modifying or deleting signals from the signal catalog can break model manifests, decoder manifests, and campaigns that reference those signals. Always check dependencies before modifying the signal catalog.

## Guardrail 8: Vehicle Data Collection Requires End-to-End Configuration
Data collection requires the entire chain to be correctly configured: signal catalog → model manifest (ACTIVE) → decoder manifest (ACTIVE) → vehicle → fleet membership → campaign (APPROVED and running). A break at any point stops data flow.

## Guardrail 9: Campaign Approval Is Required Before Data Collection
Campaigns must be explicitly approved before they start collecting data. Creating a campaign does not start collection. The campaign status must transition from CREATED to WAITING_FOR_APPROVAL to RUNNING.

## Guardrail 10: FleetWise Is Regional
FleetWise resources (signal catalogs, manifests, vehicles, fleets, campaigns) are regional. Resources in one region are not visible or accessible from another region. Ensure you are operating in the correct region.

## Guardrail 11: Vehicle Names Must Be Unique Within Account
Vehicle names are unique identifiers within an account and region. You cannot create two vehicles with the same name. Vehicle names are used as the IoT thing name for connectivity.

## Guardrail 12: Data Partitioning Affects Query Performance
S3 destination data is partitioned by vehicle name and time. Timestream data uses vehicle name as a dimension. Understanding the partitioning scheme is essential for efficient data queries and cost management.
