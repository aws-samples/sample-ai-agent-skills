---
title: "D2 — Internet Gateway Detached"
description: "Diagnose internet connectivity failures caused by missing or detached IGW"
status: active
severity: CRITICAL
triggers:
  - "No internet access"
  - "IGW detached"
  - "Public instances unreachable"
owner: devops-agent
objective: "Restore internet gateway attachment and connectivity"
context: "An IGW must be created AND attached to the VPC. Route tables must have 0.0.0.0/0 → IGW. Instances need public IPs (auto-assign or EIP). Only one IGW per VPC."
---

## Phase 1 — Triage

MUST:
- Check if an IGW exists and is attached: `aws ec2 describe-internet-gateways --filters Name=attachment.vpc-id,Values=<vpc-id>`
- If no IGW found, check if one exists but is detached
- Verify route table has 0.0.0.0/0 → IGW
- Verify instances have public IPs or EIPs

SHOULD:
- Check CloudTrail for IGW detach/delete events
- Verify the route target matches the correct IGW ID
- Check if NAT gateway is also affected (NAT needs IGW in its subnet)

MAY:
- Check if IaC deployment removed the IGW
- Verify only one IGW is attached (only one allowed per VPC)

## Common Issues

- symptoms: "All internet access lost across VPC"
  diagnosis: "IGW was detached or deleted from the VPC."
  resolution: "Reattach or create a new IGW and attach to the VPC."

- symptoms: "Route to IGW exists but shows blackhole"
  diagnosis: "IGW was deleted. Route entry remains but target is gone."
  resolution: "Create new IGW, attach to VPC, update route with new IGW ID."

## Safety Ratings

```
safety_ratings:
  - "describe-internet-gateways: GREEN — read-only IGW state inspection"
  - "describe-route-tables: GREEN — read-only route table inspection"
  - "describe-instances: GREEN — read-only instance metadata check"
  - "attach-internet-gateway: YELLOW — attaches IGW to VPC, recoverable by detaching"
  - "create-internet-gateway: YELLOW — creates new IGW, recoverable by deleting"
  - "create-route (add default route to IGW): YELLOW — adds route, recoverable by deleting"
  - "detach-internet-gateway: RED — detaches IGW, breaks all internet access for the VPC"
  - "delete-internet-gateway: RED — deletes IGW, breaks all internet access for the VPC"
```

## Escalation Conditions

- "Fix requires attaching or creating IGW in production VPC"
- "Fix requires modifying production VPC route tables"
- "All internet access lost across VPC — CRITICAL impact"
- "NAT gateways also affected (NAT needs IGW in its subnet)"

## Data Sensitivity

- HIGH: route table entries (expose network architecture and traffic paths)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: IGW attachment state, public IP assignments, VPC configuration

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER detach an IGW without confirming no resources depend on internet access"

## Phase 3 — Rollback

- If an IGW was attached via `attach-internet-gateway`: detach it with `aws ec2 detach-internet-gateway --internet-gateway-id <igw-id> --vpc-id <vpc-id>`
- If a new IGW was created: delete it with `aws ec2 delete-internet-gateway --internet-gateway-id <igw-id>` (must detach first)
- If a route was added to IGW: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0`
- If a route was updated to new IGW ID: replace it back with `aws ec2 replace-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <original-igw-id>`
- Document IGW ID, VPC attachment, and all route tables referencing the IGW before changes

## Output Format

```yaml
root_cause: "igw_detached — <detail>"
evidence:
  - type: internet_gateway
    content: "<IGW state and VPC attachment>"
severity: CRITICAL
mitigation:
  immediate: "Attach or create IGW for the VPC"
  long_term: "Protect IGW with SCPs or IAM policies to prevent accidental deletion"
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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
