# AWS Transfer Family Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Transfer Family issue.

## Guardrail 1: Endpoint Types Have Fundamentally Different Capabilities
PUBLIC endpoints get an AWS-managed hostname with IPs that can change. They do NOT support security groups or Elastic IPs directly. VPC endpoints create ENIs in your VPC subnets with static private IPs — you can attach Elastic IPs for static public IPs and security groups for IP whitelisting. VPC_ENDPOINT (legacy) uses AWS PrivateLink and is only accessible from within the VPC or peered networks. Never recommend security groups for PUBLIC endpoints.

## Guardrail 2: Authentication Modes Are Mutually Exclusive Per Server
A Transfer Family server uses either SERVICE_MANAGED or a custom identity provider (AWS_DIRECTORY_SERVICE, AWS_LAMBDA, or API_GATEWAY). You cannot mix authentication modes on the same server. Service-managed stores SSH public keys directly on the user resource. Custom IdP delegates authentication to a Lambda function or API Gateway that must return a specific JSON response format including Role and HomeDirectory.

## Guardrail 3: S3 and EFS Backends Have Different Feature Sets
S3 backends support logical directory mappings, session policies (ScopeDownPolicy), object tagging via S3 APIs, and virtually unlimited storage. EFS backends require POSIX profiles (Uid/Gid) on each user, support only physical home directories (not logical mappings), and respect POSIX file permissions. Never recommend logical directory mappings for EFS-backed servers.

## Guardrail 4: Logical vs Physical Directory Mappings Are Not Interchangeable
Logical mappings (HomeDirectoryType=LOGICAL) create a chroot-like virtual filesystem from Entry/Target pairs. Users can only see and access mapped paths. Physical mappings (HomeDirectoryType=PATH) set a single home directory path without chroot — users may navigate above their home directory if IAM permissions allow. Logical mappings require at least one mapping with Entry "/" as the root.

## Guardrail 5: Host Key Changes Cause Client-Side Warnings
SFTP clients cache the server's host key fingerprint on first connection. If you replace a Transfer Family server, change the host key, or migrate to a new server, clients will see a "host key mismatch" or "man-in-the-middle" warning and may refuse to connect. Always export the host key from the old server and import it to the new server. Each server can have multiple host keys — the most recently added active key is the one presented to clients.

## Guardrail 6: Passive FTP Requires Explicit IP Configuration Behind NAT/NLB
FTP and FTPS in Transfer Family use passive mode. The server sends its IP address in the PASV/EPSV response for the client to connect to the data channel. If the server is behind a NAT gateway or NLB, the PASV response contains the private IP by default, which the client cannot reach. You MUST set PassiveIp in ProtocolDetails to the public-facing IP (NLB IP or Elastic IP). Without this, FTP data transfers will fail even though the control connection succeeds.

## Guardrail 7: Each Protocol Has Different Port Requirements
SFTP uses port 22 only. FTPS uses port 21 (control) and ports 8192-8200 (passive data). FTP uses port 21 (control) and ports 8192-8200 (passive data). When enabling multiple protocols on a server, ALL required ports must be open in security groups (VPC endpoint) or NACLs. A common mistake is opening only port 22 when FTPS is also enabled, causing FTPS connections to fail.

## Guardrail 8: Security Policies Control Cipher Compatibility
Transfer Family security policies define which cryptographic algorithms are allowed for each protocol. TransferSecurityPolicy-2024-01 is the most restrictive (strongest ciphers only). Older clients may not support the ciphers in restrictive policies and will fail to connect with a "no matching cipher" error. Always check the client's supported ciphers against the server's security policy before changing policies.

## Guardrail 9: Session Policies Are Evaluated WITH the IAM Role — Both Must Allow
S3 session policies (ScopeDownPolicy) do not grant permissions — they restrict the permissions already granted by the user's IAM role. The effective permissions are the intersection of the IAM role policy and the session policy. If the IAM role allows s3:PutObject but the session policy does not include s3:PutObject, the upload will be denied. Always check both policies when debugging access denied errors.

## Guardrail 10: The IAM Role Trust Policy Must Trust transfer.amazonaws.com
The IAM role used by Transfer Family users must have a trust policy that allows the transfer.amazonaws.com service principal to assume it. Without this, user authentication succeeds but all file operations fail with access denied. This is a common issue when reusing roles originally created for other services (e.g., Lambda, EC2).

## Guardrail 11: VPC Endpoint Security Groups Apply to the ENI, Not the Server
Security groups for VPC endpoint-type servers are attached to the VPC endpoint's Elastic Network Interfaces (ENIs), not to the Transfer Family server resource itself. When troubleshooting connectivity, inspect the security groups on the ENIs in the specified subnets. Inbound rules must allow traffic on the protocol ports (22 for SFTP, 21 and 8192-8200 for FTP/FTPS) from the client IP ranges.

## Guardrail 12: Structured Logging Must Be Explicitly Enabled and Requires a Log Group
Structured logging to CloudWatch is NOT enabled by default. You must configure StructuredLogDestinations on the server with a CloudWatch Logs log group ARN. The server's IAM logging role must have permissions to create log streams and put log events in the specified log group. Without structured logging, you only get basic CloudWatch metrics — no detailed authentication or file operation logs.
