# AWS Entity Resolution Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Entity Resolution issue.

## Guardrail 1: Entity Resolution Is NOT Glue ETL
Entity Resolution is a separate service for matching and linking records across datasets. It uses Glue Data Catalog for input metadata but is NOT a Glue ETL job. APIs use `entityresolution:` prefix, not `glue:`.

## Guardrail 2: Schema Mappings Reference Glue Data Catalog Tables
Schema mappings point to Glue Data Catalog tables, NOT raw S3 files. You must create Glue tables (via crawler or manually) before creating schema mappings. The mapping defines which table columns correspond to Entity Resolution attributes.

## Guardrail 3: Rule-Based and ML Matching Are Different Approaches
Rule-based matching requires you to define explicit match rules (e.g., exact match on email, fuzzy match on name). ML matching uses AWS pre-trained models and doesn't require rule definition. Choose based on your data quality and matching requirements.

## Guardrail 4: ML Matching Models Are NOT Customizable
ML matching uses AWS-managed pre-trained models. You cannot train custom models, adjust model parameters, or bring your own ML model. If ML matching quality is insufficient, use rule-based matching with custom rules.

## Guardrail 5: Provider Services Require Third-Party Subscriptions
ID mapping with provider services (LiveRamp, TransUnion) requires active subscriptions with those providers. Entity Resolution facilitates the integration but doesn't include the provider subscription. Provider-specific configuration is needed.

## Guardrail 6: Entity Resolution Does NOT Modify Source Data
Matching and ID mapping produce output files in S3. Source data in Glue tables remains unchanged. Output contains match IDs, confidence scores, and matched record references. Source records are never modified.

## Guardrail 7: Output Goes to S3 with IAM Role Permissions
Job output is written to S3. The workflow's IAM role must have s3:PutObject on the output location. The role also needs glue:GetTable and glue:GetDatabase for reading input schema mappings.

## Guardrail 8: Matching Workflows Process All Input Records
Matching workflows process all records from all input schema mappings in a single job. You cannot selectively process subsets. For incremental matching, you need to manage input data partitioning externally.

## Guardrail 9: Match Rules Use Specific Comparison Types
Rule-based matching supports specific comparison types: EXACT, FUZZY, and PROVIDER. Each rule specifies which attributes to compare and the comparison type. Multiple rules create an OR condition — a match on ANY rule produces a match.

## Guardrail 10: Confidence Scores Range from 0 to 1
ML matching produces confidence scores between 0 and 1. Higher scores indicate stronger matches. You can set a confidence threshold to filter results. The threshold is configured on the matching workflow.

## Guardrail 11: Schema Mapping Attributes Have Specific Types
Entity Resolution defines standard attribute types: NAME, ADDRESS, EMAIL_ADDRESS, PHONE_NUMBER, UNIQUE_ID, etc. Input columns must be mapped to these types. Incorrect type mapping degrades match quality.

## Guardrail 12: Workflows Are Regional Resources
Matching workflows, ID mapping workflows, and schema mappings are regional. They exist in a single AWS region. For multi-region matching, create workflows in each region separately.
