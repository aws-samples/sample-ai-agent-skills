# C1 â€” FinSpace Managed kdb Cluster Issues

## Symptoms
- Cluster creation fails
- Cluster not responding to queries
- Cluster performance poor

## Triage Commands

```bash
aws finspace list-kx-clusters --environment-id <env-id> --max-results 20
aws finspace get-kx-cluster --environment-id <env-id> --cluster-name <name>
aws cloudwatch get-metric-statistics --namespace AWS/FinSpace --metric-name CPUUtilization --dimensions Name=ClusterName,Value=<name> --start-time <start> --end-time <end> --period 300 --statistics Average
```

## Cluster Types
| Type | Purpose |
|------|---------|
| HDB | Historical database (disk-based) |
| RDB | Real-time database (memory-based) |
| GATEWAY | Query routing across clusters |
| GP | General purpose mixed workload |

## Common Causes
1. **Wrong cluster type** â€” Using HDB for real-time or RDB for historical
2. **Node type too small** â€” Insufficient memory/CPU for workload
3. **Initialization script error** â€” q script fails during startup
4. **Database not attached** â€” Cluster not configured with database
5. **Network issues** â€” Cluster can't reach other clusters or services

## Resolution
- Choose correct cluster type for workload
- Size nodes based on data volume and query complexity
- Test initialization scripts locally before deploying
- Attach databases to cluster configuration
- Verify network connectivity between clusters

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — FinSpace manages financial datasets and kdb analytical environments |
| Data Risk | CRITICAL — Contains regulated financial data subject to compliance requirements |
| Recovery Complexity | HIGH — kdb cluster recreation and data reload can be time-consuming |
| Blast Radius | HIGH — Environment issues affect all datasets and analytical workloads |

## Escalation Conditions

- FinSpace environment stuck in CREATING or DELETING state beyond expected timeframe
- kdb cluster failures causing analytical workload disruptions
- Dataset access control changes blocking authorized users from financial data
- Changeset import failures preventing data updates for time-sensitive analytics
- KMS key issues preventing encryption/decryption of regulated financial data
- VPC or Transit Gateway connectivity failures isolating kdb environments

## Data Sensitivity

- Datasets contain regulated financial data (market data, trading records, risk metrics)
- Access control configurations reveal organizational data governance structure
- kdb database contents may include proprietary trading algorithms and models
- Superuser credentials grant unrestricted environment access — treat as highly privileged
- Changeset data may contain pre-release financial information subject to regulations

## PROHIBITED Actions

- NEVER suggest deleting a FinSpace environment without confirming all datasets are backed up
- NEVER modify access control policies without compliance team approval for financial data
- NEVER expose financial dataset contents in logs or troubleshooting output
- NEVER change kdb cluster configurations during active analytical workloads
- NEVER remove superuser access without ensuring alternative admin access exists
- NEVER delete changesets containing regulatory-required historical financial data

## Phase 3 — Rollback

- If environment configuration was changed: revert to previous settings and verify kdb cluster connectivity
- If dataset access was modified: restore previous permission sets and verify user access
- If kdb cluster was misconfigured: recreate cluster with previous configuration and reload databases
- If changeset import failed: retry with corrected data or revert to previous changeset version
- If VPC configuration was changed: restore Transit Gateway and subnet settings for kdb connectivity
- Document all rollback actions with environment ID, dataset names, and compliance audit references
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
