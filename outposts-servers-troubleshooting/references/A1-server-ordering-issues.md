---
title: "A1 — Server Ordering Issues"
description: "Diagnose AWS Outposts Server ordering and delivery problems"
status: active
severity: HIGH
triggers: ["outpost order stuck", "server not delivered", "order status unknown", "cannot order outpost server"]
owner: devops-agent
objective: "Resolve Outposts Server ordering issues including site configuration, order status, and delivery delays"
context: "Outposts Servers must have a site created before ordering. Orders go through validation, preparation, shipping, and installation phases."
---

## Phase 1 — Triage

MUST:
- List existing Outpost sites: `aws outposts list-sites --query 'Sites[*].{SiteId:SiteId,Name:Name,City:City,OperatingAddressCity:OperatingAddressCity}'`
- List existing orders: `aws outposts list-orders --query 'Orders[*].{OrderId:OrderId,OutpostId:OutpostId,Status:Status,OrderType:OrderType}'`
- Check Outpost lifecycle status: `aws outposts list-outposts --query 'Outposts[*].{Id:OutpostId,Name:Name,LifeCycleStatus:LifeCycleStatus}'`

SHOULD:
- Verify site configuration has correct address and operating details
- Check IAM permissions for outposts:CreateOrder and outposts:CreateOutpost
- Review AWS account status and service quotas

MAY:
- Contact AWS Support for order-specific status updates
- Review CloudTrail for CreateOrder or CreateOutpost API errors

## Phase 2 — Remediate

MUST:
- Ensure site is created with valid physical address before ordering
- Verify account has accepted the Outposts terms and conditions
- Confirm the selected server configuration is available in the target Region

SHOULD:
- Set up site networking prerequisites (power, network cabling) before delivery
- Prepare the local gateway and subnet configuration in advance

MAY:
- Use AWS Organizations to manage Outposts across multiple accounts
- Pre-configure VPC, subnets, and security groups for the Outpost

## Common Issues

- symptoms: "CreateOutpost returns ValidationException"
  diagnosis: "Site not created or site configuration incomplete."
  resolution: "Create site first: aws outposts create-site --name <name> --operating-address-city <city> --operating-address-state-or-region <state> --operating-address-country-code <code>."

## Safety Ratings

```
safety_ratings:
  - "list-sites: GREEN — read-only enumeration of Outpost sites"
  - "list-orders: GREEN — read-only order status check"
  - "list-outposts: GREEN — read-only Outpost enumeration"
  - "create-site: YELLOW — creates a site resource, non-destructive"
```

## Escalation Conditions

- Order has been in PENDING status for more than 10 business days without update
- Site validation fails repeatedly despite correct address information
- Server configuration requested is not available in the target Region
- Delivery timeline conflicts with critical project deadlines

## Data Sensitivity

- `aws outposts list-sites`: MEDIUM — reveals physical site addresses and locations
- `aws outposts list-orders`: LOW — returns order metadata and status
- `aws outposts list-outposts`: LOW — returns Outpost resource metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER cancel an existing Outposts order without explicit approval from the account owner
- NEVER modify site address information without verifying the correct physical location
- NEVER share site physical address details in logs or public channels
- NEVER assume order delays are AWS issues without checking account-side prerequisites

## Phase 3 — Rollback

1. If a site was created with incorrect information, update it: `aws outposts update-site --site-id <id> --name <corrected-name>`
2. If an order was placed in error, contact AWS Support immediately — orders cannot be cancelled via API
3. If VPC/subnet pre-configuration was done for the wrong Outpost, delete the unused subnets
4. Document the ordering timeline and any issues for future reference
5. Verify site prerequisites (power, network, physical space) are still valid

## Output Format

```yaml
root_cause: "server_ordering — <specific_cause>"
evidence:
  - type: order_status
    content: "<order details and status>"
severity: HIGH
mitigation:
  immediate: "Resolve ordering blocker"
  long_term: "Document site prerequisites checklist"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
