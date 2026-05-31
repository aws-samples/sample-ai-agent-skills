# AWS Backup Gateway Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Backup Gateway issue.

## Guardrail 1: Gateway Is an On-Premises Appliance
Backup Gateway is a virtual appliance deployed in the on-premises VMware environment. It runs as a VM on ESXi. Issues may be on-premises infrastructure problems (network, storage, compute) rather than AWS service issues. Always verify on-premises health first.

## Guardrail 2: Dual Network Connectivity Required
The gateway requires network connectivity in two directions: to the VMware vCenter/ESXi hosts (on-premises) AND to AWS service endpoints (internet/Direct Connect). Failure in either direction causes different symptoms. Always test both paths.

## Guardrail 3: vCenter Credentials Are Stored Encrypted
vCenter credentials are stored encrypted on the gateway. If vCenter passwords are rotated, the hypervisor connection breaks silently. The gateway does not alert on credential expiry — backup jobs simply fail. Always check credential validity after password rotations.

## Guardrail 4: Changed Block Tracking (CBT) Is Critical
Backup Gateway uses VMware Changed Block Tracking for incremental backups. If CBT is disabled, reset, or unavailable (e.g., after Storage vMotion), the next backup becomes a full backup, consuming significantly more time and bandwidth.

## Guardrail 5: Restore Creates a New VM
Restore operations create a NEW virtual machine in the target vCenter. They do NOT overwrite or modify the original VM. The restored VM requires post-restore configuration (network, DNS, IP address) before it is production-ready.

## Guardrail 6: VM Discovery Is Periodic
VM discovery from vCenter is periodic, not real-time. New VMs, renamed VMs, or VMs moved between folders may take time to appear in the discovered VM list. Do not assume a VM is missing from the environment based on discovery results alone.

## Guardrail 7: Bandwidth Is Shared Across Operations
Gateway bandwidth is shared across all concurrent backup and restore operations. Running multiple operations simultaneously reduces throughput for each individual operation. Plan backup windows to avoid bandwidth contention.

## Guardrail 8: Gateway Requires Minimum Resources
The gateway VM requires minimum CPU, memory, and disk resources. Under-provisioned gateways experience performance degradation, timeouts, and backup failures. Verify gateway VM resource allocation matches AWS recommendations.

## Guardrail 9: Time Synchronization Is Critical
The gateway must have accurate time synchronization (NTP) with both the vCenter environment and AWS. Time skew causes authentication failures, certificate validation errors, and backup scheduling issues.

## Guardrail 10: Proxy Configuration Affects All Traffic
If the gateway is configured to use a proxy, ALL traffic (including vCenter communication) routes through the proxy. Misconfigured proxy settings can break both AWS connectivity and hypervisor connections simultaneously.

## Guardrail 11: Gateway Software Updates Are Automatic
Gateway software updates are applied automatically by AWS. Updates may cause brief connectivity interruptions. Do not assume a brief gateway offline period is a failure — check if an update was applied.

## Guardrail 12: Backup Vault Policies Affect Backup Storage
Backup data is stored in AWS Backup vaults. Vault access policies, encryption settings, and retention policies affect backup operations. A backup job can succeed at the gateway level but fail at the vault level due to policy restrictions.
