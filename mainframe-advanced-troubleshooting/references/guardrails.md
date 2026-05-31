# AWS Mainframe Modernization Advanced Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Mainframe Modernization issue.

## Guardrail 1: Blu Age and Micro Focus Are Different Engines
Blu Age refactors COBOL to Java (automated code conversion). Micro Focus replatforms COBOL to run natively on AWS. They have different runtime environments, deployment models, and troubleshooting approaches. Never apply Blu Age guidance to Micro Focus or vice versa.

## Guardrail 2: Runtime Environments Must Be Available Before Deployment
Applications are deployed TO runtime environments. The environment must be in Available status before deployment. Environment creation can take 15-30 minutes. Do not attempt deployment to a Creating or Failed environment.

## Guardrail 3: Batch Jobs Follow Mainframe Semantics
Batch jobs in M2 map to JCL job streams. They have step-level execution, condition codes, and restart/checkpoint semantics. Do not assume serverless or container execution patterns. Job failures may be at the step level, not the job level.

## Guardrail 4: CICS Is Micro Focus Only
CICS transaction processing is only available in Micro Focus runtime environments. Blu Age converts CICS programs to Java web services. Do not troubleshoot CICS configuration in Blu Age environments.

## Guardrail 5: EBCDIC to ASCII Conversion Is Critical
Mainframe data uses EBCDIC encoding. Migration to AWS requires EBCDIC-to-ASCII conversion. Incorrect conversion causes data corruption that may not be immediately visible. Always verify data integrity after migration.

## Guardrail 6: Copybooks Define Data Structures
COBOL copybooks define record layouts for datasets. Data migration tools use copybooks to parse and convert data. Missing or incorrect copybooks cause data parsing failures. Always verify copybook accuracy before migration.

## Guardrail 7: Application Definitions Are S3-Based
M2 application definitions (JSON) and deployment artifacts are stored in S3. S3 bucket permissions, object paths, and artifact structure must be correct. S3 access issues are a common cause of deployment failures.

## Guardrail 8: Environment Engine Type Is Immutable
Once a runtime environment is created with an engine type (bluage or microfocus), it cannot be changed. To switch engines, create a new environment. Do not attempt to deploy Blu Age applications to Micro Focus environments or vice versa.

## Guardrail 9: VPC Configuration Affects Connectivity
M2 runtime environments run in VPCs. Subnet selection, security groups, and route tables affect connectivity to databases, on-premises systems, and other AWS services. Network issues are a common cause of application failures.

## Guardrail 10: Dataset Types Have Different Handling
VSAM (KSDS, ESDS, RRDS), sequential files, GDG (Generation Data Groups), and PDS (Partitioned Data Sets) each have different migration and runtime handling. Do not assume all dataset types are handled identically.

## Guardrail 11: Condition Codes Are Not Exit Codes
Mainframe condition codes (return codes) have specific meanings. CC=0 is success, CC=4 is warning, CC=8+ is error. These are NOT Unix exit codes. Do not apply Unix/Linux exit code semantics to mainframe condition codes.

## Guardrail 12: Modernization Is Iterative Not Atomic
Mainframe modernization is typically done in phases — assess, mobilize, migrate, modernize. Do not assume a single deployment completes the modernization. Hybrid operation (mainframe + AWS) is normal during migration.
