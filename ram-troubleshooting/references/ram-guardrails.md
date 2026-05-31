# AWS Resource Access Manager Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS RAM issue.

## Guardrail 1: Organization Sharing Does NOT Require Invitations
When sharing resources within an AWS Organization (with RAM organization sharing enabled), resources are automatically available to member accounts. No invitation is sent or needed. Invitations are ONLY required when sharing with accounts outside the organization or when organization sharing is not enabled.

## Guardrail 2: Not All Resource Types Support RAM Sharing
Only specific AWS resource types can be shared through RAM. Common shareable types include VPC subnets, Transit Gateways, Route 53 Resolver rules, License Manager configurations, Aurora DB clusters, and CodeBuild projects. Check supported types with: aws ram list-resource-types. Attempting to share unsupported types fails silently or with an error.

## Guardrail 3: allowExternalPrincipals Controls External Sharing
The allowExternalPrincipals flag on a resource share controls whether accounts outside the organization can be added as principals. When false, only organization member accounts can access shared resources. This is a security control — changing it to true enables external sharing. SCPs can also restrict this setting.

## Guardrail 4: RAM Permissions Are NOT IAM Policies
RAM permissions define what actions participants can perform on shared resources. They are separate from IAM policies. Both RAM permissions AND IAM policies must allow an action for it to succeed. RAM permissions are attached to resource shares, not to IAM principals. Each resource type has a default managed permission.

## Guardrail 5: Shared VPC Subnets Have Specific Limitations
Participant accounts can launch resources (EC2, RDS, etc.) into shared subnets but CANNOT modify the subnet, VPC, route tables, NACLs, or internet gateways. The owner account retains full control. Security groups are NOT shared — each account creates and manages its own security groups. Participants cannot see or modify other accounts' resources in the shared subnet.

## Guardrail 6: Resource Share Status Must Be ACTIVE
Resource shares go through lifecycle states: PENDING → ACTIVE, or FAILED. Only ACTIVE shares provide access to shared resources. Shares can also be DELETING or DELETED. Check status with: aws ram get-resource-shares. If a share is not ACTIVE, troubleshoot the creation or update that caused the non-active state.

## Guardrail 7: Organization Sharing Must Be Explicitly Enabled
RAM organization sharing requires a one-time enablement in the management account: aws ram enable-sharing-with-aws-organization. Without this, sharing within the organization requires invitations like external sharing. This creates a service-linked role for RAM in the organization.

## Guardrail 8: Cross-Region Sharing Is NOT Supported for Most Types
RAM shares resources within the same AWS region only. You cannot share a VPC subnet in us-east-1 with an account expecting to use it in us-west-2. Transit Gateway peering is a separate feature from RAM sharing. Some global resources may appear to work cross-region but are actually region-specific.

## Guardrail 9: Deleting a Resource Share Removes Access Immediately
When a resource share is deleted, all participant accounts lose access to the shared resources immediately. Resources launched by participants (e.g., EC2 instances in shared subnets) continue to run but may lose connectivity or functionality. Always verify no critical workloads depend on a share before deleting.

## Guardrail 10: Principal Types Affect Sharing Behavior
RAM supports multiple principal types: AWS account IDs, organization ARNs, OU ARNs, and IAM role/user ARNs (for some resource types). The principal type determines the scope of sharing. Organization and OU principals share with all accounts in that scope. Account-level principals share with the specific account only.

## Guardrail 11: Permission Versions Must Be Explicitly Set
RAM managed permissions can have multiple versions. When a new version is released, existing resource shares continue using the old version until explicitly updated. Use: aws ram list-permission-versions to check available versions. Updating to a new version may change the actions available to participants.

## Guardrail 12: Resource Ownership Does NOT Transfer
RAM sharing does NOT transfer ownership of resources. The owner account retains full control and is responsible for the resource. Participants get the permissions defined by the RAM permission attached to the share. The owner can revoke access at any time by removing the principal or deleting the share.
