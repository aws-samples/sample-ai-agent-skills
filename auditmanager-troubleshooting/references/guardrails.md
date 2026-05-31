# AWS Audit Manager Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Audit Manager issue.

## Guardrail 1: AWS Config Is Required for Automated Evidence
Most automated controls rely on AWS Config rules for evidence collection. Without Config enabled and recording, automated evidence collection fails silently. Verify Config: aws config describe-configuration-recorders.

## Guardrail 2: Standard Frameworks Cannot Be Modified
Built-in frameworks (CIS, PCI DSS, SOC 2, HIPAA, etc.) are read-only. To customize controls, create a custom framework based on the standard one. Never suggest modifying standard frameworks directly.

## Guardrail 3: Evidence Collection Is Scheduled
Evidence is collected on a schedule (daily for Config-based, varies for others). It is NOT real-time. New resources may take up to 24 hours to appear in evidence. Do not assume missing evidence means non-compliance.

## Guardrail 4: Assessment Reports Require S3 Bucket
Assessment reports are stored in an S3 bucket configured in Audit Manager settings. The bucket must exist, have proper permissions, and allow Audit Manager to write. Without a configured bucket, report generation fails.

## Guardrail 5: Delegation Requires IAM Permissions
Delegates need IAM permissions for auditmanager:* scoped to their delegated assessments. Without proper IAM, delegates cannot view or update their assigned control sets.

## Guardrail 6: Audit Manager Is Regional
Assessments and evidence are regional. Resources in us-east-1 are not included in eu-west-1 assessments. For multi-region compliance, create assessments in each region.

## Guardrail 7: Control Types Determine Evidence Source
Controls can be automated (Config rules, Security Hub, CloudTrail, API calls) or manual. The control type determines how evidence is collected. Automated controls need the corresponding service enabled.

## Guardrail 8: Security Hub Integration Is Optional
Some controls use Security Hub findings as evidence. This requires Security Hub to be enabled with the relevant standards. Without Security Hub, these controls show no automated evidence.

## Guardrail 9: Manual Evidence Must Be Uploaded
Manual controls require evidence to be uploaded by users (documents, screenshots, attestations). Audit Manager cannot automatically collect evidence for manual controls.

## Guardrail 10: Assessment Scope Defines What's Evaluated
Assessment scope (AWS accounts, services) determines which resources are evaluated. Resources outside the scope are not assessed. Verify scope matches compliance requirements.

## Guardrail 11: Evidence Folders Organize by Control
Evidence is organized into folders per control within each control set. Each folder contains evidence items collected over time. Folder structure follows the framework's control hierarchy.

## Guardrail 12: Audit Manager Costs Per Assessment
Audit Manager charges per resource assessment. Costs depend on the number of resources evaluated and the frequency of evidence collection. Large assessments with many resources can incur significant costs.
