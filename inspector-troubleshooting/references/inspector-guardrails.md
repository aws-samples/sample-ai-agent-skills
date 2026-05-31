# Amazon Inspector Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Inspector issue.

## Guardrail 1: Inspector v2 Is a Different Service from Inspector Classic
Inspector v2 (inspector2 API) is the current service with agentless and agent-based scanning. Inspector Classic (inspector API) is the legacy service requiring assessment targets and templates. They have different APIs, pricing, and capabilities. Do not use Classic API commands for v2 issues. Inspector v2 does not use assessment templates.

## Guardrail 2: EC2 Scanning Requires SSM Agent and Managed Instance Status
Inspector cannot scan EC2 instances without the SSM agent running and the instance appearing as a managed instance in Systems Manager. The instance needs an IAM role with AmazonSSMManagedInstanceCore policy. Check SSM agent status with describe-instance-information. Instances without SSM agent show as "NOT_COVERED" in Inspector coverage.

## Guardrail 3: ECR Scanning Is Triggered on Image Push and CVE Updates
When ECR scanning is enabled, Inspector automatically scans images when pushed to ECR repositories. It also rescans existing images when new CVEs are published. Scanning is per-repository. Enhanced scanning (Inspector) replaces basic ECR scanning. Both cannot be active simultaneously on the same repository.

## Guardrail 4: Lambda Scanning Has Two Modes — Code and Standard
Lambda code scanning analyzes function code for vulnerabilities (injection, hardcoded credentials). Lambda standard scanning checks runtime versions and layers for known CVEs. These are separate scan types that must be enabled independently. Code scanning is available in fewer regions than standard scanning.

## Guardrail 5: Findings Are Regional — No Automatic Cross-Region Visibility
Inspector findings exist only in the region where the resource is located. To see findings across regions, enable Inspector in each region. Use Security Hub with cross-region aggregation for centralized visibility. Do not assume findings from one region appear in another.

## Guardrail 6: Suppression Hides Findings, It Does Not Remediate Vulnerabilities
Suppression rules filter findings from the active view. Suppressed findings still exist, still count toward quotas, and can be viewed with the suppressed filter. Suppression is for accepted risks, not remediation. Always remediate vulnerabilities when possible. Suppression rules use filter criteria (severity, resource type, title, etc.).

## Guardrail 7: Inspector Pricing Is Per Resource Scanned, Not Per Finding
EC2 scanning: per instance per month. ECR scanning: per image initially scanned + rescan. Lambda scanning: per function per month. Finding count does not affect cost. Disabling scanning for unused resources reduces cost. Check pricing page for current rates.

## Guardrail 8: Coverage Gaps Indicate Resources Not Being Scanned
Coverage statistics show how many resources are scanned vs total. Gaps occur when EC2 instances lack SSM agent, ECR repositories have basic scanning instead of enhanced, or Lambda functions are in unsupported runtimes. Use list-coverage with NOT_COVERED filter to identify gaps.

## Guardrail 9: Delegated Admin Manages Inspector for the Organization
Only one account can be the delegated admin for Inspector per organization. The delegated admin can enable/disable scanning for member accounts, view findings across accounts, and manage suppression rules. Changing delegated admin requires disabling the current one first.

## Guardrail 10: Finding Severity Levels Follow CVSS Scoring
Inspector uses CVSS v3 scores mapped to severity levels: CRITICAL (9.0-10.0), HIGH (7.0-8.9), MEDIUM (4.0-6.9), LOW (0.1-3.9), INFORMATIONAL (0.0). Inspector may adjust severity based on exploit availability and network exposure. The Inspector score may differ from the NVD CVSS score.

## Guardrail 11: Scan Triggers Differ by Resource Type
EC2: scanned on SSM agent inventory changes, new CVE publications, and instance state changes. ECR: scanned on image push and new CVE publications. Lambda: scanned on function creation/update and new CVE publications. Manual rescans are not supported — scanning is event-driven.

## Guardrail 12: Resource Tags Can Filter Scanning Scope
Inspector supports tag-based inclusion/exclusion for EC2 scanning. Tags can be used to exclude development instances or include only production resources. Tag filters are configured at the account level. ECR and Lambda scanning does not support tag-based filtering — it's all-or-nothing per resource type.
