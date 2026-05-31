---
title: "C1 — Cloud Map Instance Registration Issues"
description: "Diagnose instance registration failures"
status: active
severity: MEDIUM
triggers:
  - "instance registration failed"
  - "cannot register instance"
  - "instance not appearing"
  - "register instance error"
owner: devops-agent
objective: "Identify and resolve instance registration issues in Cloud Map"
context: "Instance registration adds endpoints to a Cloud Map service. For DNS namespaces, this creates Route 53 records. Registration can fail due to quota limits, invalid attributes, or IAM permissions."
---

## Phase 1 — Triage

MUST:
- List current instances: `aws servicediscovery list-instances --service-id <service-id>`
- Check operation status: `aws servicediscovery get-operation --operation-id <operation-id>`
- Verify service exists: `aws servicediscovery get-service --id <service-id>`
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names servicediscovery:RegisterInstance`
- Check CloudTrail for registration events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RegisterInstance --max-results 10`

SHOULD:
- Check instance quota for the service
- Verify instance attributes are valid (AWS_INSTANCE_IPV4, AWS_INSTANCE_PORT, etc.)
- For DNS services, verify IP address format

MAY:
- Check for duplicate instance IDs
- Verify Route 53 record creation for DNS namespaces

## Phase 2 — Remediate

MUST:
- Provide required attributes (AWS_INSTANCE_IPV4 for A records, AWS_INSTANCE_IPV4 + AWS_INSTANCE_PORT for SRV)
- Use unique instance IDs within the service
- Grant servicediscovery:RegisterInstance permission

SHOULD:
- Request quota increase if hitting instance limits
- Add meaningful custom attributes for API discovery filtering
- Implement registration retry logic for transient failures

MAY:
- Automate instance registration in deployment pipelines
- Set up registration monitoring

## Common Issues

- symptoms: "RegisterInstance returns quota exceeded"
  diagnosis: "Service has reached the maximum number of instances (default 1,000)."
  resolution: "Request a quota increase or deregister unused instances."

- symptoms: "Registration succeeds but DNS record not created"
  diagnosis: "Instance attributes missing required fields for the DNS record type."
  resolution: "Include AWS_INSTANCE_IPV4 for A records or add AWS_INSTANCE_PORT for SRV."

- symptoms: "Registration fails with invalid parameter"
  diagnosis: "Instance ID contains invalid characters or attributes are malformed."
  resolution: "Use valid instance IDs and correctly formatted attribute values."

## Output Format

```yaml
root_cause: "instance_registration_issue — <specific_cause>"
evidence:
  - type: operation_status
    content: "<registration operation details>"
  - type: service_config
    content: "<service DNS configuration>"
  - type: instance_attributes
    content: "<provided attributes>"
severity: MEDIUM
mitigation:
  immediate: "Fix attributes or permissions and retry registration"
  long_term: "Automate instance lifecycle management"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Provide required attributes | GREEN |
| Use unique instance IDs | GREEN |
| Grant servicediscovery:RegisterInstance permission | YELLOW |
| Request quota increase | GREEN |
| Add meaningful custom attributes | GREEN |
| Implement registration retry logic | GREEN |

## Escalation Conditions

- Fix requires modifying production instance registrations
- Fix involves changing IAM permissions for service discovery

## Data Sensitivity

- **MEDIUM**: instance registrations expose service endpoints, IP addresses, and port information
- Custom attributes may contain environment-specific configuration details

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If instances were registered incorrectly, deregister them: `aws servicediscovery deregister-instance --service-id <service-id> --instance-id <instance-id>`
2. If IAM permissions were modified, revert to previous policy
3. If quota increase was requested, no rollback needed
4. Verify rollback by confirming `aws servicediscovery list-instances` shows expected registrations

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
