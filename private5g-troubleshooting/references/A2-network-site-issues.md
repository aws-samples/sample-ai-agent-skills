---
title: "A2 — Network Site Issues"
description: "Diagnose AWS Private 5G network site problems"
status: active
severity: HIGH
triggers: ["network site failed", "site not active", "CreateNetworkSite error"]
owner: devops-agent
objective: "Resolve network site creation and configuration issues"
context: "Network sites represent physical locations with radio units. Each site must be associated with a network."
---

## Phase 1 — Triage

MUST:
- List network sites: `aws private-networks list-network-sites --network-arn <arn>`
- Get site details: `aws private-networks get-network-site --network-site-arn <arn>`
- Check site status — should be ACTIVE or CREATED

SHOULD:
- Verify the parent network is active: `aws private-networks get-network --network-arn <arn>`
- Review CloudTrail for site-related errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateNetworkSite --max-results 10`

MAY:
- Check if hardware orders are associated with the site: `aws private-networks list-orders --network-arn <arn>`

## Phase 2 — Remediate

MUST:
- Ensure parent network is in AVAILABLE status before creating sites
- Provide valid site configuration with correct availability zone and subnet
- Fix IAM permissions if access denied

SHOULD:
- Verify subnet has available IP addresses for ENIs
- Confirm the availability zone matches your physical location planning

MAY:
- Create multiple sites for multi-location deployments

## Safety Ratings

```
safety_ratings:
  - "list-network-sites: GREEN — read-only enumeration"
  - "get-network-site: GREEN — read-only site details"
  - "CreateNetworkSite: YELLOW — creates site and may trigger hardware order"
  - "list-orders: GREEN — read-only order status"
```

## Escalation Conditions

- Site creation fails with internal errors after all validations pass
- Site stuck in CREATED status and not transitioning to ACTIVE
- Hardware associated with site is not being delivered
- Multiple sites failing simultaneously across the same network

## Data Sensitivity

- `aws private-networks list-network-sites`: LOW — returns site metadata
- `aws private-networks get-network-site`: LOW — returns site configuration details
- Site configuration: MEDIUM — includes subnet and availability zone information

## Prohibited Actions

- NEVER delete a network site with active radio units without proper decommissioning
- NEVER modify site configuration while devices are actively connected
- NEVER create sites in availability zones without verifying physical location alignment
- NEVER ignore pending hardware orders when troubleshooting site issues

## Phase 3 — Rollback

1. If a site was created in error, check for associated hardware orders first
2. Remove any network resources from the site: `aws private-networks list-network-resources --network-arn <arn>`
3. Delete the site only if no hardware is deployed: `aws private-networks delete-network-site --network-site-arn <arn>`
4. Verify the parent network is unaffected by the site deletion
5. Document the rollback and update deployment plans

## Common Issues

- symptoms: "Network site stuck in CREATED status"
  diagnosis: "Hardware order may be pending or site configuration incomplete."
  resolution: "Check order status and verify all site configuration parameters are correct."

- symptoms: "CreateNetworkSite returns SubnetNotFoundException"
  diagnosis: "Specified subnet does not exist or is in a different availability zone."
  resolution: "Verify subnet ID and ensure it exists in the correct AZ."

## Output Format

```yaml
root_cause: "network_site — <specific_cause>"
evidence:
  - type: site_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix site configuration"
  long_term: "Validate site parameters before creation"
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
