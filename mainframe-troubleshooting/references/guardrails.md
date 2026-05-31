# AWS Mainframe Modernization Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Mainframe Modernization issue.

## Guardrail 1: Two Engines — Micro Focus and Blu Age
Micro Focus engine replatforms mainframe applications (runs COBOL as-is on AWS). Blu Age engine refactors mainframe code to modern Java. They have different capabilities, deployment models, and use cases. Do not mix engine-specific advice.

## Guardrail 2: Runtime Environment Must Exist Before Deployment
Applications must be deployed to a runtime environment. The environment must be in Available state before deployment. Environment creation takes several minutes and provisions compute resources.

## Guardrail 3: Application Definition Specifies the Engine
The application definition (JSON) specifies which engine (micro-focus or blu-age) the application uses. The application can only be deployed to environments running the matching engine.

## Guardrail 4: Data Sets Must Be Available Before Batch Jobs
Batch jobs reference data sets by name. Data sets must be imported or created in the application before batch jobs can use them. Missing data sets cause job failures with data set not found errors.

## Guardrail 5: CICS/IMS Is Micro Focus Engine Only
CICS and IMS transaction processing is only available with the Micro Focus engine. Blu Age refactors these to modern equivalents (REST APIs, etc.). Do not suggest CICS/IMS configuration for Blu Age applications.

## Guardrail 6: Deployment Is Separate from Application Creation
Creating an application defines its configuration. Deploying the application pushes it to a runtime environment. These are separate steps. An application can exist without being deployed.

## Guardrail 7: Application Artifacts Come from S3
Application binaries (COBOL load modules, Blu Age artifacts) must be uploaded to S3 before creating the application. The application definition references the S3 location.

## Guardrail 8: Environment Sizing Affects Performance
Runtime environment instance type and count affect application performance. Under-provisioned environments cause slow batch jobs and transaction timeouts. Monitor CPU and memory metrics.

## Guardrail 9: VPC Configuration Required for Environments
Runtime environments are deployed in a VPC with specified subnets and security groups. Network configuration affects connectivity to databases, file systems, and other services.

## Guardrail 10: Batch Job Scheduling Uses JCL or API
Batch jobs can be submitted via JCL (Micro Focus) or API calls. Scheduling can use EventBridge rules or external schedulers. The M2 service doesn't have built-in scheduling.

## Guardrail 11: Data Set Import Supports Specific Formats
Data set import supports specific mainframe data formats. VSAM, sequential, and GDG data sets have different import procedures. Verify format compatibility before import.

## Guardrail 12: Application Logs Go to CloudWatch
Application logs (COBOL DISPLAY, Blu Age application logs) are sent to CloudWatch Logs. Check CloudWatch log groups for application-level errors. Log group names follow the pattern /aws/m2/<environment-id>.
