# AWS Firewall Manager Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Firewall Manager issue.

## Guardrail 1: Organizations with All Features Is Required
AWS Firewall Manager requires AWS Organizations with ALL features enabled. Consolidated billing only is NOT sufficient. If Organizations is not enabled or only has consolidated billing, FMS APIs will fail. Verify with: aws organizations describe-organization --query 'Organization.FeatureSet'.

## Guardrail 2: Only One FMS Administrator Account Per Organization
Only one AWS account can be the Firewall Manager administrator at a time. This can be the management account or a delegated administrator. Changing the admin account removes all existing FMS policies. Verify the current admin with: aws fms get-admin-account.

## Guardrail 3: Auto-Remediation Has Limitations
Auto-remediation (RemediationEnabled=true) does not fix all violations instantly. For WAF policies, it creates web ACLs but may not remove conflicting rules. For security group policies, it can add/remove rules but may not handle complex dependencies. For Network Firewall, it deploys endpoints but VPC route table updates may require manual action. Always verify remediation status.

## Guardrail 4: FMS-Managed WAF Web ACLs Are Overwritten
Firewall Manager creates and manages WAF web ACLs in member accounts. Manual changes to FMS-managed web ACLs (prefixed with FMManagedWebACLV2-) are overwritten when FMS re-evaluates the policy. Custom rules should be added through the FMS policy configuration, not directly to the managed web ACL.

## Guardrail 5: Shield Advanced Costs $3,000/Month Per Account
Shield Advanced subscription is $3,000/month per organization (not per account when using consolidated billing under Organizations). However, data transfer charges for DDoS response still apply. FMS Shield policies can enable Shield Advanced on member accounts. Do not enable Shield Advanced without understanding the cost implications.

## Guardrail 6: Security Group Policies Have Two Distinct Types
FMS security group policies come in two types: USAGE policies (enforce baseline security group rules) and AUDIT policies (report non-compliant security groups). Usage policies can auto-remediate by applying required rules. Audit policies only report findings. Do not confuse the two types — they have fundamentally different behaviors and configurations.

## Guardrail 7: Network Firewall Deployment Creates Resources in Member Accounts
FMS Network Firewall policies deploy firewall endpoints (ENIs) in member account VPCs. This creates resources that incur costs (endpoint hours + data processing). Deployment can take several minutes per VPC. Deleting the FMS policy removes the deployed firewalls. Route table updates for Network Firewall traffic inspection may require additional configuration.

## Guardrail 8: DNS Firewall Policies Use Rule Group Priority
FMS DNS Firewall policies distribute DNS Firewall rule groups to member account VPCs. Rule groups have priorities that determine evaluation order. FMS-managed rule groups coexist with manually created rule groups. Domain lists (AWS-managed or custom) define which domains to allow, deny, or alert on. Priority conflicts can cause unexpected behavior.

## Guardrail 9: Policy Scope Determines Which Resources Are Protected
FMS policy scope is defined by resource tags, specific accounts, organizational units (OUs), and resource types. If a resource doesn't match the scope criteria, the policy does not apply. Incorrect scope configuration is the #1 cause of policies not protecting expected resources. Verify scope with: aws fms get-policy --policy-id <id> --query 'Policy.ResourceTags'.

## Guardrail 10: Cross-Account Policy Enforcement Has Propagation Delay
When FMS policies are created or updated, changes propagate to member accounts asynchronously. Compliance evaluation may take minutes to hours depending on the number of accounts and resources. Do not assume immediate compliance after policy creation. Check compliance status with: aws fms list-compliance-status --policy-id <id>.

## Guardrail 11: Compliance Evaluation Timing Varies by Policy Type
WAF policies evaluate compliance when web ACLs are checked against resources. Security group policies evaluate when security group rules are compared. Network Firewall policies evaluate when firewall endpoints are verified. Shield policies evaluate when Shield Advanced protection is checked. Each type has different evaluation intervals and triggers.

## Guardrail 12: FMS-Managed Resources Follow Naming Conventions
FMS creates resources with specific naming prefixes: FMManagedWebACLV2- for WAF web ACLs, FMS- for security groups, and similar patterns for other resource types. These naming conventions help identify FMS-managed resources. Do not rename or manually modify FMS-managed resources as changes will be overwritten.
