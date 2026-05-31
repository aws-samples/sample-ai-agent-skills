# AWS Organizations Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Organizations issue.

## Guardrail 1: Management Account Is EXEMPT from SCPs
The management account (formerly master account) is NEVER affected by Service Control Policies, regardless of what SCPs are attached to the root, OUs, or the account itself. This is by design to prevent organizational lockout. Only member accounts are subject to SCPs. Never test SCP behavior from the management account — it will always appear to have full access.

## Guardrail 2: SCPs Are Deny Filters, NOT Permission Grants
SCPs define the maximum permissions boundary for member accounts. An SCP Allow statement does NOT grant permissions — it only permits the action to potentially be allowed by IAM policies. An SCP Deny statement blocks the action regardless of any IAM policy. The effective permissions are the INTERSECTION of SCP allows and IAM grants. Think of SCPs as a filter, not a faucet.

## Guardrail 3: Account Email Must Be Globally Unique
Every AWS account requires a globally unique email address across ALL AWS accounts worldwide. You cannot reuse an email from a recently closed account until closure is fully processed (90+ days). Use email aliases (user+tag@domain.com) or distribution lists for creating multiple accounts. The email is used for root account access and cannot be easily changed.

## Guardrail 4: Account Closure Has a 90-Day Recovery Period
After closing an account via `close-account`, the account enters a suspended state for 90 days. During this period, the account can be reopened by contacting AWS Support or using the root credentials. After 90 days, the account is permanently closed and all resources are irreversibly deleted. Billing charges may still accrue during the suspension period for resources that cannot be immediately stopped.

## Guardrail 5: Delegated Administrator Is Limited to Specific Services
Not all AWS services support delegated administration. Each service defines its own delegation model and capabilities. Typically, a maximum of 5 accounts can be delegated administrators for a single service. The management account must explicitly enable delegation for each service. Delegated admins can only perform service-specific organizational operations, not general Organizations management.

## Guardrail 6: OU Nesting Maximum Is 5 Levels
Organizational Units can be nested up to 5 levels deep below the root. The root itself is the top level. This means the deepest OU path is: Root → OU1 → OU2 → OU3 → OU4 → OU5. Attempting to create a 6th level fails. Plan the OU hierarchy carefully because restructuring requires moving accounts one at a time, and each move changes the account's effective SCPs immediately.

## Guardrail 7: Maximum 5 SCPs Per Target
Each target (root, OU, or individual account) can have a maximum of 5 SCPs attached directly. The default FullAWSAccess SCP counts as one of the 5. Inherited SCPs from parent OUs do NOT count against this limit on the child. However, all inherited and directly attached SCPs are evaluated together. The maximum SCP document size is 5,120 bytes.

## Guardrail 8: Policy Inheritance Is Cumulative — All Ancestors Apply
SCPs are inherited from ALL parent OUs up to the root. The effective SCP for an account is the intersection of all SCPs at every level from root to the account. A Deny at ANY level blocks the action for all descendants — there is no override. Moving an account between OUs changes its effective SCPs IMMEDIATELY. Always evaluate the full OU path when troubleshooting SCP issues.

## Guardrail 9: Consolidated Billing Shares RI and Savings Plan Benefits
Reserved Instances and Savings Plans purchased in any member account are shared across the entire organization by default. This sharing can be disabled per linked account. Volume discounts (e.g., S3 tiered pricing) are calculated across the organization's aggregate usage. Cost allocation tags must be activated at the management account level to appear in billing reports.

## Guardrail 10: Removing an Account Requires Root Email Access
To remove a member account from the organization (making it standalone), the account must have: a valid payment method configured, root email access for verification, and an appropriate support plan. Accounts created via Organizations API may not have root credentials set up — you must use the "forgot password" flow with the account email to establish root access before removal.

## Guardrail 11: Organization Feature Set Cannot Be Downgraded
Organizations can operate in "consolidated billing only" or "all features" mode. Once upgraded to "all features" (which enables SCPs, tag policies, etc.), the organization CANNOT be downgraded back to "consolidated billing only." The only way to revert is to delete the organization and recreate it, which requires removing all member accounts first.

## Guardrail 12: SCP Document Size Limit Is 5,120 Bytes
Each SCP document has a maximum size of 5,120 bytes (approximately 5 KB). This includes all whitespace and formatting. Complex SCPs with many conditions or resource ARNs can easily exceed this limit. Use multiple SCPs (up to 5 per target) to work around the size limit. Minify JSON (remove whitespace) to maximize usable space.
