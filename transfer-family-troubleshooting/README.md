# Transfer Family Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Transfer Family problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Transfer Family when the console alone isn't enough — server configuration analysis, authentication debugging (service-managed and custom IdP), connectivity issues, S3 and EFS backend problems, directory mapping configuration, performance optimization, host key management, managed workflow failures, and protocol-specific issues (SFTP, FTPS, FTP).

### Activate When

- Server creation or startup failures
- Endpoint configuration issues (PUBLIC, VPC, VPC_ENDPOINT)
- Protocol misconfiguration (SFTP, FTPS, FTP)
- Service-managed authentication failures
- Custom Identity Provider (Lambda/API Gateway) errors
- SSH key authentication problems
- Connection timeouts or refused connections
- Firewall or NAT traversal issues
- Passive FTP data channel failures
- S3 backend access denied or permission errors
- EFS backend mount or permission issues
- Logical directory mapping not working
- Home directory configuration problems
- Slow transfer speeds or throughput limits
- Large file transfer failures or timeouts
- Host key mismatch warnings
- Encryption or security policy issues
- Managed workflow execution failures
- Post-upload processing errors
- Structured logging not appearing in CloudWatch

---

## Skill Structure

```
transfer-family-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-server-creation-failures.md
    ├── A2-endpoint-issues.md
    ├── A3-protocol-configuration.md
    ├── B1-service-managed-auth.md
    ├── B2-custom-idp-failures.md
    ├── B3-ssh-key-issues.md
    ├── C1-connection-failures.md
    ├── C2-firewall-nat.md
    ├── C3-passive-ftp.md
    ├── D1-s3-backend-issues.md
    ├── D2-efs-backend-issues.md
    ├── E1-directory-mapping.md
    ├── E2-home-directory-config.md
    ├── F1-throughput-limits.md
    ├── F2-large-file-transfers.md
    ├── G1-host-key-management.md
    ├── G2-encryption-in-transit.md
    ├── H1-managed-workflow-failures.md
    ├── H2-post-upload-processing.md
    ├── Z1-general-troubleshooting.md
    ├── transfer-family-guardrails.md
    └── transfer-family-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Server** | A1–A3 | Server creation failures, endpoint issues, protocol configuration |
| **B — Authentication** | B1–B3 | Service-managed auth, custom IdP failures, SSH key issues |
| **C — Connectivity** | C1–C3 | Connection failures, firewall/NAT, passive FTP |
| **D — Storage** | D1–D2 | S3 backend issues, EFS backend issues |
| **E — Directory** | E1–E2 | Directory mapping, home directory configuration |
| **F — Performance** | F1–F2 | Throughput limits, large file transfers |
| **G — Security** | G1–G2 | Host key management, encryption in transit |
| **H — Workflows** | H1–H2 | Managed workflow failures, post-upload processing |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/transfer-family-guardrails.md` covering endpoint type differences, authentication mode exclusivity, S3 vs EFS backend capabilities, logical vs physical directory mappings, host key continuity, passive FTP IP configuration, protocol port requirements, security policy cipher compatibility, session policy evaluation, IAM role trust policy requirements, VPC endpoint security groups, and structured logging configuration.

---

## Investigation Workflow

1. **Triage** — List servers, check server state, collect CloudWatch metrics for file/byte counts
2. **Deep Dive** — Examine identity provider, endpoint details, protocol details, workflow configuration, user directory mappings
3. **Detailed** — Inspect workflow executions, structured logs, host keys, security groups, VPC endpoints, CloudTrail events

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `transfer:*`, `s3:GetBucketPolicy`, `s3:ListBucket`, `efs:DescribeFileSystems`, `efs:DescribeMountTargets`, `ec2:DescribeSecurityGroups`, `ec2:DescribeVpcEndpoints`, `ec2:DescribeNetworkInterfaces`, `lambda:GetFunction`, `apigateway:GET`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `logs:FilterLogEvents`, `iam:GetRole`
- Structured logging enabled on Transfer Family servers (recommended)
- CloudWatch metrics enabled

---

## Usage Examples

```
# List all Transfer Family servers
aws transfer list-servers

# Get full server configuration
aws transfer describe-server --server-id s-1234567890abcdef0

# Test custom identity provider
aws transfer test-identity-provider --server-id s-1234567890abcdef0 --user-name testuser --user-password "password"

# Check user configuration and directory mappings
aws transfer describe-user --server-id s-1234567890abcdef0 --user-name testuser

# List workflow executions
aws transfer list-executions --workflow-id w-1234567890abcdef0

# Check structured logs
aws logs filter-log-events --log-group-name /aws/transfer/s-1234567890abcdef0 --start-time 1704067200000
```

---

## License

MIT-0
