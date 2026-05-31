# AWS Application Composer Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Application Composer issue.

## Guardrail 1: SAM Templates Are Not Plain CloudFormation
Application Composer generates SAM templates with Transform: AWS::Serverless-2016-10-31. SAM resources (AWS::Serverless::Function, AWS::Serverless::Api) are NOT valid CloudFormation without the transform. Do not suggest deploying SAM templates with plain CloudFormation deploy commands.

## Guardrail 2: Canvas State Can Diverge from Template File
The visual canvas and the underlying template file can become out of sync, especially after manual template edits outside Application Composer. Always verify the actual template file content rather than relying on canvas appearance.

## Guardrail 3: Project Sync Overwrites Local Files
Project sync writes Application Composer changes directly to local files. If the local project has uncommitted changes, sync can overwrite them without warning. Always commit or stash changes before enabling sync.

## Guardrail 4: Connections Generate Hidden IAM Policies
Canvas connections between resources automatically generate IAM policy statements and environment variables. Removing a connection from the canvas does NOT remove the previously generated IAM statements from the template. Manual cleanup is required.

## Guardrail 5: Template Import Silently Drops Unsupported Resources
When importing a template, Application Composer silently drops resource types or properties it cannot render on the canvas. The resources still exist in the template but are not visible or editable on the canvas. Always check the template source after import.

## Guardrail 6: Local Development Requires Docker
SAM CLI local invoke and local start-api require Docker to emulate Lambda runtimes. Missing Docker, incorrect Docker socket permissions, or Docker Desktop not running causes build and invoke failures that may appear as Application Composer issues.

## Guardrail 7: IDE Extension Requires Valid AWS Credentials
The VS Code AWS Toolkit extension for Application Composer requires valid AWS credentials. Expired tokens, missing profiles, or SSO session timeouts cause extension errors that look like Application Composer bugs. Always verify credentials first.

## Guardrail 8: CloudFormation Stack Limits Apply
Application Composer-generated templates are subject to CloudFormation limits: 500 resources per stack, 51,200 byte template body, 1 MB template in S3. Large applications may hit these limits during deployment.

## Guardrail 9: Resource Properties Are Schema-Driven
Application Composer resource property panels are generated from CloudFormation resource schemas. If a property is missing from the panel, it may need to be added manually in the template source. Not all properties are exposed in the visual editor.

## Guardrail 10: Deployment Uses CloudFormation Under the Hood
Application Composer deployments go through CloudFormation (or SAM CLI which wraps CloudFormation). Deployment failures are CloudFormation failures. Check CloudFormation stack events, not Application Composer UI, for deployment error details.

## Guardrail 11: Custom Resources Have Limited Canvas Support
Custom::* resources and AWS::CloudFormation::CustomResource render as generic cards on the canvas. Property editing is limited to raw JSON/YAML. Custom resource logic (Lambda-backed) must be configured outside the visual editor.

## Guardrail 12: Multi-Stack Applications Require Manual Coordination
Application Composer works with a single template at a time. Multi-stack architectures with cross-stack references (Exports/Imports) require manual coordination. Application Composer does not manage cross-stack dependencies automatically.
