# AWS License Manager Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any License Manager issue.

## Guardrail 1: License Configurations vs Received Licenses
License configurations are self-managed tracking rules you create. Received licenses come from ISVs via Marketplace or direct grants. They have different management workflows and APIs.

## Guardrail 2: Hard Limits Block Launches
When a license configuration has hard limit enforcement enabled, exceeding the license count blocks new resource launches (EC2 instances, RDS instances). Soft limits only generate CloudWatch alerts without blocking.

## Guardrail 3: Grants Must Be Accepted
License grants must be explicitly accepted by the recipient account before they take effect. A grant in PENDING_ACCEPT status does not provide license access. Grants can also be rejected.

## Guardrail 4: Host Resource Groups Require Auto-Placement
Host resource groups manage dedicated host allocation for license tracking. Auto-placement must be enabled on dedicated hosts for automatic instance placement. Without it, instances must specify a host ID.

## Guardrail 5: Linux Subscription Discovery Requires SSM
Linux subscription discovery uses Systems Manager inventory to find RHEL, SUSE, and Ubuntu Pro subscriptions. Instances must have SSM Agent installed and running with proper IAM permissions.

## Guardrail 6: User Subscriptions Need Active Directory
User-based subscriptions (Microsoft Visual Studio, Office) require Active Directory integration. This needs a specific VPC configuration with directory service connectivity.

## Guardrail 7: Cross-Account Sharing Uses RAM
Cross-account license sharing uses AWS Resource Access Manager (RAM) under the hood. Organization sharing must be enabled in RAM settings for organization-wide sharing.

## Guardrail 8: Automated Discovery Has Scope Limits
Automated discovery uses SSM inventory to find software installations. It only discovers software on instances with SSM Agent. On-premises servers require SSM hybrid activation.

## Guardrail 9: License Rules Are Per-Configuration
License counting rules (vCPU, instance, core, socket) are set per license configuration. Different software products may need different counting methods. Do not apply one product's rules to another.

## Guardrail 10: Organization Settings Require Management Account
License Manager organization settings (cross-account discovery, sharing) must be configured from the management account or delegated admin. Member accounts cannot modify organization settings.

## Guardrail 11: Dedicated Host Licensing Is Instance-Type Specific
Dedicated hosts are tied to specific instance families. License tracking on dedicated hosts counts by the host's instance type capacity. You cannot mix instance families on a single host.

## Guardrail 12: License Manager Is Regional
License configurations and associations are regional. Cross-region license tracking requires configurations in each region. Received licenses (ISV grants) may have region restrictions.
