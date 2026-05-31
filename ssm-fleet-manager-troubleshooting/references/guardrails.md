# AWS Systems Manager Fleet Manager Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Fleet Manager issue.

## Guardrail 1: SSM Agent Version Matters
Fleet Manager requires SSM Agent 3.0+ for advanced features (remote desktop, file system, performance counters). Basic node management works with older agents. Always check agent version: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,Agent:AgentVersion}'`.

## Guardrail 2: IAM Instance Profile Is Required
EC2 instances must have an IAM instance profile with `AmazonSSMManagedInstanceCore` managed policy (or equivalent permissions). Missing profiles are the most common cause of nodes not appearing. Custom policies must include ssm, ssmmessages, and ec2messages permissions.

## Guardrail 3: Hybrid Nodes Use Different Registration
On-premises and non-EC2 nodes use hybrid activations with `mi-` prefixed IDs. They require activation codes and IDs, not IAM instance profiles. Hybrid nodes have registration limits per activation and expiration dates.

## Guardrail 4: VPC Endpoints Required for Private Subnets
Instances in private subnets without NAT gateways require VPC endpoints for `com.amazonaws.<region>.ssm`, `com.amazonaws.<region>.ssmmessages`, and `com.amazonaws.<region>.ec2messages`. Missing any one of these three endpoints causes connectivity failures.

## Guardrail 5: Fleet Manager Is Regional
Fleet Manager displays nodes per region. Nodes connect to SSM in the region configured in the SSM Agent. Multi-region fleets require checking each region separately. Cross-region visibility requires AWS Systems Manager Explorer or Resource Data Sync.

## Guardrail 6: Remote Desktop Tunnels Through SSM
Fleet Manager remote desktop uses Session Manager tunneling. It does NOT require inbound security group rules for RDP (3389) or SSH (22). It requires the SSM Agent, proper IAM permissions, and Session Manager plugin. Network issues are SSM connectivity issues, not firewall issues.

## Guardrail 7: Performance Counters Are Windows-Only
Fleet Manager performance counters are available only for Windows managed nodes. For Linux nodes, use CloudWatch Agent or SSM Run Command with system monitoring commands. Do not suggest performance counters for Linux instances.

## Guardrail 8: Patch Visibility Requires Patch Baseline
OS patching visibility in Fleet Manager requires a configured patch baseline and at least one scan or install operation via Patch Manager. Nodes without patch operations show no patch data — this does not mean they are compliant.

## Guardrail 9: SSM Agent Auto-Update Is Optional
SSM Agent auto-update can be enabled via `aws ssm update-service-setting`. It is not enabled by default on all platforms. Manual updates may be required for older AMIs or custom images. Check current version before assuming auto-update is active.

## Guardrail 10: Connection Status vs Ping Status
`PingStatus` in describe-instance-information reflects the last SSM Agent heartbeat. `get-connection-status` checks real-time connectivity. A node with PingStatus=Online may still fail connections if the agent crashed after the last heartbeat.

## Guardrail 11: Managed Instance Tier Affects Limits
The standard tier allows up to 1,000 on-premises instances per account per region at no cost. The advanced tier removes this limit but incurs per-instance charges. Check the tier setting before troubleshooting registration limit issues.

## Guardrail 12: File System Access Requires Session Manager
Fleet Manager file system browsing uses Session Manager under the hood. It requires the same prerequisites: SSM Agent 2.3.68+, IAM permissions for ssm:StartSession, and Session Manager plugin. File operations are logged in Session Manager audit logs.
