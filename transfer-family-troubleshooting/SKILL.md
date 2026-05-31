---
name: transfer-family-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Transfer Family problems
  by analyzing server configurations, authentication settings, storage backends,
  directory mappings, connectivity, and following structured runbooks.
  Activate when: server creation failures, endpoint configuration issues,
  protocol misconfiguration, service-managed auth failures, custom IdP errors,
  SSH key problems, connection failures, firewall/NAT issues, passive FTP
  problems, S3 backend errors, EFS backend errors, directory mapping issues,
  home directory configuration, throughput limits, large file transfer failures,
  host key management, encryption in transit, managed workflow failures,
  post-upload processing errors, or the user says something is wrong with
  Transfer Family without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Transfer Family, S3, EFS, IAM, Lambda,
  API Gateway, CloudWatch, CloudTrail, EC2 (security groups, VPC endpoints),
  and optionally Route 53 and ACM permissions.
---

# Transfer Family Diagnostics

## When to use

Any Transfer Family investigation where the console alone is insufficient — server configuration analysis, authentication debugging, connectivity troubleshooting, storage backend issues, directory mapping problems, performance optimization, security configuration, workflow failures, or protocol-specific issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws transfer list-servers --query 'Servers[*].{ServerId:ServerId,State:State,EndpointType:EndpointType,Protocols:Protocols}'
aws transfer describe-server --server-id <server-id>
aws transfer list-users --server-id <server-id>
aws transfer describe-user --server-id <server-id> --user-name <username>
aws cloudwatch get-metric-statistics --namespace AWS/Transfer --metric-name FilesIn --dimensions Name=ServerId,Value=<server-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Transfer --metric-name BytesIn --dimensions Name=ServerId,Value=<server-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws transfer describe-server --server-id <server-id> --query 'Server.IdentityProviderDetails'
aws transfer describe-server --server-id <server-id> --query 'Server.EndpointDetails'
aws transfer describe-server --server-id <server-id> --query 'Server.ProtocolDetails'
aws transfer describe-server --server-id <server-id> --query 'Server.WorkflowDetails'
aws transfer describe-user --server-id <server-id> --user-name <username> --query 'User.HomeDirectoryMappings'
aws transfer list-executions --workflow-id <workflow-id> --query 'Executions[*].{ExecutionId:ExecutionId,Status:Status}'
```

### Step 3 — Detailed investigation

```
aws transfer describe-execution --execution-id <execution-id> --workflow-id <workflow-id>
aws transfer describe-server --server-id <server-id> --query 'Server.StructuredLogDestinations'
aws transfer list-host-keys --server-id <server-id>
aws transfer describe-host-key --server-id <server-id> --host-key-id <host-key-id>
aws ec2 describe-security-groups --group-ids <sg-id>
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=transfer.amazonaws.com --max-results 10
aws logs filter-log-events --log-group-name /aws/transfer/<server-id> --start-time <epoch-ms> --end-time <epoch-ms>
```

Read `references/transfer-family-guardrails.md` before concluding on any Transfer Family issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-servers` | List all Transfer Family servers |
| `describe-server` | Full server configuration details |
| `list-users` | List users on a server |
| `describe-user` | User configuration, home directory, role |
| `list-executions` | Check workflow execution status |
| `describe-execution` | Workflow execution details and errors |
| `list-host-keys` | Server host key inventory |
| `describe-host-key` | Host key details and fingerprint |
| `test-identity-provider` | Test custom IdP authentication |
| `describe-security-policy` | Security policy cipher/protocol details |
| `list-connectors` | List SFTP connectors |
| `describe-connector` | Connector configuration details |

## Gotchas: Transfer Family

- Endpoint types: PUBLIC endpoints get an AWS-managed IP that can change. VPC endpoints get static private IPs via ENIs. VPC_ENDPOINT (legacy) uses AWS PrivateLink. For static public IPs, use VPC endpoint type with an internet-facing NLB or Elastic IPs on the ENIs.
- Authentication modes: SERVICE_MANAGED stores SSH keys in the Transfer Family service. Custom IdP uses a Lambda function or API Gateway to authenticate. You cannot mix service-managed and custom IdP on the same server. Custom IdP must return a JSON response with Role, HomeDirectory, and optionally Policy and PublicKeys.
- S3 vs EFS backend: S3 supports logical directory mappings, session policies, and object tagging. EFS requires a POSIX profile (Uid/Gid) on each user. EFS does not support logical directory mappings — only physical home directories. S3 does not support POSIX file permissions.
- Directory mappings: Logical directory mappings (HomeDirectoryType=LOGICAL) map virtual paths to S3 or EFS physical paths. The Entry field is the virtual path the user sees; the Target is the actual S3 bucket/prefix or EFS path. Users are chroot-jailed to their mapped directories. Physical home directories (HomeDirectoryType=PATH) set a single home directory without chroot.
- Host key management: SFTP clients cache the server host key fingerprint. If you replace a server or import a new host key, clients will see a "host key mismatch" warning. Export the host key from the old server and import it to the new one to avoid disruption. Each server can have multiple host keys — the most recently added active key is used.
- Passive mode FTP: FTP and FTPS use passive mode by default in Transfer Family. The server sends its endpoint IP in the PASV response. For VPC endpoints behind a NAT or NLB, the PassiveIp must be set in ProtocolDetails to the public-facing IP. Otherwise, clients receive a private IP they cannot reach.
- Protocol differences: SFTP runs over SSH (port 22). FTPS is FTP over TLS (port 21 for control, 8192-8200 for data). FTP is unencrypted (port 21, data 8192-8200). A single server can support multiple protocols, but each protocol has different security implications and port requirements.
- Bandwidth throttling: Transfer Family does not have built-in per-user bandwidth throttling. Throughput depends on the endpoint type, instance size (for VPC endpoints), and the backend storage. S3 supports 5 Gbps per prefix per partition. EFS throughput depends on the file system mode (bursting vs provisioned vs elastic).
- IP whitelisting: Only VPC endpoint type supports security groups for IP whitelisting. PUBLIC endpoint type does not support security groups — use S3 bucket policies or session policies to restrict by source IP. For VPC endpoints, attach security groups to the VPC endpoint ENIs.
- Structured logging: Transfer Family supports structured JSON logging to CloudWatch Logs. Enable via StructuredLogDestinations on the server. Logs include authentication events, file operations (upload, download, delete, rename), and errors. Log group format: /aws/transfer/<server-id>.
- Security policies: Security policies control the cryptographic algorithms for SFTP, FTPS, and FTP. TransferSecurityPolicy-2024-01 is the most restrictive. Older policies allow weaker ciphers. Clients must support at least one cipher in the server's security policy to connect.
- Session policies: S3 session policies (ScopeDownPolicy) restrict user access beyond the IAM role. They use IAM policy variables like ${transfer:UserName} and ${transfer:HomeBucket}. Session policies are evaluated in addition to the IAM role policy — both must allow the action.

## Anti-hallucination rules

1. Always cite specific server configurations, user settings, or API responses as evidence.
2. Endpoint types have different capabilities — PUBLIC has no security groups, VPC has static IPs. Never conflate them.
3. Custom IdP response format is strict — Role and HomeDirectory are required. Never claim optional fields are sufficient.
4. EFS does not support logical directory mappings. Never recommend logical mappings for EFS backends.
5. PassiveIp must be set for FTP/FTPS behind NAT/NLB. Never assume passive mode works automatically in all network topologies.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Server | A1-A3 | Server creation failures, endpoint issues, protocol configuration |
| B — Authentication | B1-B3 | Service-managed auth, custom IdP failures, SSH key issues |
| C — Connectivity | C1-C3 | Connection failures, firewall/NAT, passive FTP |
| D — Storage | D1-D2 | S3 backend issues, EFS backend issues |
| E — Directory | E1-E2 | Directory mapping, home directory configuration |
| F — Performance | F1-F2 | Throughput limits, large file transfers |
| G — Security | G1-G2 | Host key management, encryption in transit |
| H — Workflows | H1-H2 | Managed workflow failures, post-upload processing |
| Z — Catch-All | Z1 | General troubleshooting |
