# Amazon GuardDuty Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any GuardDuty issue.

## Guardrail 1: GuardDuty Uses Independent Data Streams
GuardDuty analyzes VPC Flow Logs, DNS logs, and CloudTrail management events using its own independent data streams. You do NOT need to enable VPC Flow Logs, Route 53 DNS query logging, or CloudTrail for GuardDuty to function. GuardDuty accesses these data sources directly. Enabling them separately is recommended for your own investigation but is not a prerequisite for GuardDuty.

## Guardrail 2: Findings Are Regional with No Built-In Global View
Each AWS region has its own GuardDuty detector and generates its own findings. There is no built-in global findings view. To aggregate findings across regions, use Security Hub cross-region aggregation, export findings to a central S3 bucket, or use EventBridge rules to forward findings. Always check the correct region when investigating findings.

## Guardrail 3: Suppression Rules Hide Findings but Do Not Delete Them
Suppression rules automatically archive matching findings — they are hidden from the default findings list but NOT deleted. Suppressed findings still exist, still count toward limits, and can be viewed by filtering for archived findings. Suppression rules use filter criteria (finding type, severity, resource tags, etc.). Removing a suppression rule does not unarchive previously suppressed findings.

## Guardrail 4: S3 Protection Monitors CloudTrail S3 Data Events
S3 protection detects suspicious S3 API activity by analyzing CloudTrail S3 data events. This is a separate data source from CloudTrail management events. S3 protection has its own pricing based on the number of S3 data events analyzed. It does not require you to enable CloudTrail S3 data event logging separately — GuardDuty uses its own stream.

## Guardrail 5: EKS Protection Has Two Separate Components
EKS Audit Log Monitoring analyzes Kubernetes audit logs from EKS clusters for suspicious activity. EKS Runtime Monitoring detects OS-level and container-level threats but requires the GuardDuty security agent on EKS nodes. These are separate features with separate pricing. Enabling one does not enable the other. EKS audit logs are accessed independently — you do not need to enable EKS control plane logging.

## Guardrail 6: Runtime Monitoring Requires Agent Deployment
Runtime monitoring for EC2, EKS, and ECS Fargate requires the GuardDuty security agent to be deployed on the target resources. Without the agent, no runtime findings are generated. The agent can be auto-managed (GuardDuty deploys it) or manually deployed. Auto-management uses SSM for EC2 and EKS add-ons for EKS. Coverage status shows which resources have the agent installed.

## Guardrail 7: Malware Scanning Creates EBS Snapshots with Cost Implications
GuardDuty Malware Protection scans EBS volumes by creating snapshots. These snapshots incur standard EBS snapshot pricing. Snapshots are automatically deleted after scanning. There are size limits for scanned volumes (default 1 TB) and scan timeout limits. Malware scanning is triggered by specific finding types — it does not continuously scan all volumes.

## Guardrail 8: Delegated Admin Has Full Authority Over Member Accounts
The delegated administrator can enable/disable GuardDuty features, manage findings, configure suppression rules, and manage threat intel sets for ALL member accounts. Member accounts cannot override settings configured by the delegated admin. The delegated admin can auto-enable GuardDuty for new accounts joining the organization. Only one delegated admin per organization per region.

## Guardrail 9: Findings Auto-Archive After 90 Days of Inactivity
Active findings that receive no new activity for 90 days are automatically archived. Archived findings are retained but hidden from the default view. For long-term retention, export findings to S3 using publishing destinations. The 90-day auto-archive timer resets each time the finding is updated with new activity. Archived findings cannot be unarchived manually.

## Guardrail 10: Trusted IP Lists Only Affect Certain Finding Types
Trusted IP lists prevent GuardDuty from generating findings for traffic from those IPs, but ONLY for VPC Flow Log-based and CloudTrail-based findings. Trusted IPs do NOT suppress DNS-based findings, malware findings, or runtime findings. A maximum of 6 trusted IP lists can be configured per detector. Each list can contain up to 250,000 IP addresses.

## Guardrail 11: Finding Severity Is Classified as Low, Medium, or High
GuardDuty findings have severity values from 0.1 to 8.9. Low: 0.1-3.9, Medium: 4.0-6.9, High: 7.0-8.9. Severity is assigned by GuardDuty based on the finding type and context — it cannot be manually adjusted. High-severity findings indicate active compromise or imminent threat. Always prioritize high-severity findings for immediate investigation.

## Guardrail 12: Suspending vs Disabling a Detector Are Different
Suspending a detector stops GuardDuty from processing data and generating new findings, but retains existing findings and configuration. Disabling (deleting) a detector permanently removes all findings, configurations, and settings. Suspension is reversible — disabling is not. Always suspend rather than disable if you plan to re-enable GuardDuty later.
