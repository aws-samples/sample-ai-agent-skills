---
title: "F1 — High Availability"
description: "Diagnose and resolve CloudHSM high availability and cluster redundancy issues"
status: active
severity: CRITICAL
triggers:
  - "single HSM"
  - "HSM failover"
  - "cluster availability"
  - "AZ failure"
  - "HSM redundancy"
owner: devops-agent
objective: "Ensure CloudHSM cluster meets high availability requirements with proper multi-AZ redundancy"
context: "CloudHSM high availability requires at least 2 HSMs in different Availability Zones. The client SDK automatically handles failover between HSMs. A single-HSM cluster is a single point of failure — if the HSM fails, all cryptographic operations stop. AWS recommends at least 2 HSMs for production and 3 for critical workloads. HSMs in a cluster automatically synchronize keys and users."
---

## Phase 1 — Triage

MUST:
- Check cluster HSM count and distribution: `aws cloudhsmv2 describe-clusters --filters clusterIds=<cluster-id> --query 'Clusters[0].Hsms[*].{ID:HsmId,State:State,AZ:AvailabilityZone,IP:EniIp}'`
- Verify all HSMs are in ACTIVE state
- Check if HSMs are distributed across different AZs
- Verify the client SDK is configured with all HSM IPs: `cat /opt/cloudhsm/etc/cloudhsm_client.cfg`
- Check cluster backup status: `aws cloudhsmv2 describe-backups --filters clusterIds=<cluster-id> --max-results 5`

SHOULD:
- Test failover by verifying the client can operate if one HSM IP is unreachable
- Check CloudWatch metrics for HSM health and latency
- Verify automatic backups are enabled and recent

MAY:
- Review the cluster's AZ distribution against the application's AZ deployment
- Check if the client SDK version supports automatic HSM discovery
- Assess the throughput requirements against the number of HSMs

## Phase 2 — Remediate

MUST:
- If only 1 HSM exists: add a second HSM in a different AZ: `aws cloudhsmv2 create-hsm --cluster-id <cluster-id> --availability-zone <az>`
- If an HSM is not ACTIVE: delete and reprovision: `aws cloudhsmv2 delete-hsm --cluster-id <cluster-id> --hsm-id <hsm-id>` then create a new one
- If client config has only one HSM IP: update to include all HSM IPs and restart the client
- If backups are missing: verify automatic backups are enabled (they are by default)

SHOULD:
- Add a third HSM for critical production workloads
- Ensure HSMs are in AZs that match the application deployment
- Verify key and user synchronization after adding new HSMs: `listUsers` and `findKey` on each HSM

MAY:
- Implement CloudWatch alarms for HSM state changes
- Set up automated HSM replacement using Lambda and CloudWatch Events
- Document the HA architecture and failover procedures

## Common Issues

- symptoms: "Application crypto operations fail when one HSM goes down"
  diagnosis: "Single-HSM cluster or client not configured for failover."
  resolution: "Add a second HSM in a different AZ. Update client config with all HSM IPs."

- symptoms: "New HSM added but client doesn't use it"
  diagnosis: "Client configuration not updated with the new HSM IP."
  resolution: "Update cloudhsm_client.cfg with the new HSM IP and restart the client daemon."

- symptoms: "HSM in one AZ shows DEGRADED while others are ACTIVE"
  diagnosis: "Hardware or AZ-specific issue affecting one HSM."
  resolution: "Delete the degraded HSM and provision a new one. The cluster continues operating on remaining HSMs."

- symptoms: "Keys or users missing on newly added HSM"
  diagnosis: "Synchronization in progress or failed."
  resolution: "Wait for automatic sync (usually completes in seconds). If persistent, check HSM health."

## Output Format

```yaml
root_cause: "High availability issue — <specific_cause>"
evidence:
  - type: cluster_status
    content: "<describe-clusters output with HSM details>"
  - type: client_config
    content: "<client configuration showing HSM endpoints>"
severity: CRITICAL
mitigation:
  immediate: "Add HSMs for redundancy or fix degraded HSMs"
  long_term: "Implement multi-AZ deployment with automated health monitoring"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 adding new HSMs: YELLOW — creates billable resources"
  - "Phase 2 deleting degraded HSMs: RED — removes HSM instance"

## Escalation Conditions

escalation_conditions:
  - "Cluster is down to a single HSM in production"
  - "Multiple HSMs show DEGRADED state simultaneously"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "AZ-level failure affects HSM availability"
  - "HSM provisioning fails in all available AZs"

## Data Sensitivity

data_sensitivity:
  - command: "describe-clusters"
    sensitivity: HIGH
    contains: "HSM IDs, ENI IPs, and cluster topology"
  - command: "describe-backups"
    sensitivity: HIGH
    contains: "Backup IDs and timestamps"
  - command: "cloudhsm_client.cfg"
    sensitivity: HIGH
    contains: "HSM IP addresses and certificate paths"

## Prohibited Actions

prohibited_actions:
  - "NEVER reduce a production cluster to a single HSM intentionally"
  - "NEVER delete an HSM without verifying other HSMs are ACTIVE"
  - "NEVER skip backup verification before HSM replacement"
  - "NEVER place all HSMs in the same Availability Zone for production"
  - "NEVER disable automatic backups on production clusters"

## Phase 3 — Rollback

- Before deleting any HSM, verify the cluster has at least one other ACTIVE HSM
- If a newly added HSM causes issues, delete it — the cluster continues on existing HSMs
- If client configuration changes break connectivity, restore from backup config
- If HSM replacement fails, restore the cluster from the most recent backup as a last resort
- Verify all applications maintain connectivity after any cluster topology changes
