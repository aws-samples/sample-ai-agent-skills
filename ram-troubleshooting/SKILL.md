---
name: ram-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Resource Access Manager
  (RAM) problems by analyzing resource share creation, invitations, acceptance,
  supported resources, sharing failures, cross-account access, organization
  sharing, auto-sharing, managed permissions, custom permissions, access denied
  errors, resource visibility issues, and following structured runbooks.
  Activate when: resource share creation failures, invitation not received,
  acceptance errors, unsupported resource types, sharing failures, cross-account
  resource access issues, organization sharing not working, auto-sharing
  configuration problems, managed permission errors, custom permission issues,
  access denied when accessing shared resources, shared resources not visible,
  or the user says something is wrong with RAM without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with RAM, Organizations, IAM, CloudTrail,
  and service-specific permissions for shared resources. Some operations
  require organization management account permissions.
---

# AWS Resource Access Manager (RAM) Diagnostics

## When to use

Any AWS RAM investigation where the console alone is insufficient — resource share creation failures, invitation issues, cross-account resource access, organization sharing configuration, permission management, or resource visibility problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ram get-resource-shares --resource-owner SELF --query 'resourceShares[*].{Name:name,Arn:resourceShareArn,Status:status,AllowExternal:allowExternalPrincipals}'
version: "1.0.0"
last_updated: "2025-04-12"
aws ram list-resources --resource-owner SELF --query 'resources[*].{Arn:arn,Type:type,Status:status}'
aws ram list-principals --resource-owner SELF --query 'principals[*].{Id:id,ResourceShareArn:resourceShareArn}'
aws organizations describe-organization --query 'Organization.{Id:Id,FeatureSet:FeatureSet}'
```

### Step 2 — Domain deep dive

```
aws ram get-resource-share-invitations --query 'resourceShareInvitations[*].{Arn:resourceShareInvitationArn,ShareArn:resourceShareArn,Status:status,Sender:senderAccountId}'
aws ram list-permissions --query 'permissions[*].{Arn:arn,Name:name,ResourceType:resourceType,IsResourceTypeDefault:isResourceTypeDefault}'
version: "1.0.0"
last_updated: "2025-04-12"
aws ram get-permission --permission-arn <permission-arn>
aws ram list-resource-types --query 'resourceTypes[*].{ResourceType:resourceType,ServiceName:serviceName}'
version: "1.0.0"
last_updated: "2025-04-12"
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ram.amazonaws.com --max-results 20
aws ram list-pending-invitation-resources --resource-share-invitation-arn <invitation-arn>
aws ram get-resource-share-associations --association-type PRINCIPAL --resource-share-arns <share-arn>
aws ram get-resource-share-associations --association-type RESOURCE --resource-share-arns <share-arn>
```

Read `references/ram-guardrails.md` before concluding on any RAM issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ram get-resource-shares` | List and check resource share status |
| `ram list-resources` | List resources in a share |
| `ram list-principals` | List principals with access |
| `ram get-resource-share-invitations` | Check invitation status |
| `ram list-permissions` | List available RAM permissions |
| `ram get-permission` | Get permission details |
| `ram list-resource-types` | List shareable resource types |
| `ram get-resource-share-associations` | Check resource and principal associations |
| `ram list-pending-invitation-resources` | List resources in pending invitations |

## Gotchas: AWS RAM

- RAM sharing within Organizations does NOT require invitations. Resources shared within an organization are automatically available to member accounts. Invitations are only needed for sharing with accounts outside the organization.
- Not all AWS resource types support RAM sharing. Check supported types with list-resource-types. Common shareable resources include VPC subnets, Transit Gateways, Route 53 Resolver rules, License Manager configurations, and Aurora DB clusters.
- allowExternalPrincipals controls sharing outside the organization. When set to false, only accounts within the organization can be added as principals. This is a security control that prevents accidental external sharing.
- RAM permissions define what actions principals can perform on shared resources. Each resource type has a default managed permission. Custom permissions can be created for fine-grained access control. Permissions are attached to resource shares.
- Shared VPC subnets have specific limitations. Participant accounts can launch resources into shared subnets but cannot modify the subnet, VPC, or route tables. The owner account retains full control. Security groups are not shared — each account manages its own.
- Resource share status must be ACTIVE for sharing to work. Shares can be in PENDING, ACTIVE, FAILED, or DELETING states. Only ACTIVE shares provide access to shared resources.
- Organization sharing requires enabling RAM in Organizations. Use: aws ram enable-sharing-with-aws-organization. This is a one-time setup in the management account.
- Cross-region sharing is NOT supported for most resource types. Resources can only be shared within the same region. Transit Gateway sharing is an exception that works across regions.
- Deleting a resource share removes access immediately. All principals lose access to shared resources when the share is deleted. This can cause service disruptions if resources are in use.

## Anti-hallucination rules

1. Always cite specific resource share ARNs, invitation status, or API responses as evidence.
2. Not all resource types support RAM sharing. Never assume a resource type is shareable.
3. Organization sharing doesn't need invitations. Never suggest sending invitations within an org.
4. RAM permissions are NOT IAM policies. Never confuse the two permission systems.
5. Cross-region sharing is not supported for most types. Never assume cross-region works.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 18 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Sharing | A1-A3 | Resource share creation, invitation, acceptance |
| B — Resources | B1-B3 | Supported resources, sharing failures, cross-account |
| C — Organization | C1-C2 | Organization sharing, auto-sharing |
| D — Permissions | D1-D2 | Managed permissions, custom permissions |
| E — Troubleshooting | E1-E2 | Access denied, resource not visible |
| Z — Catch-All | Z1 | General troubleshooting |
