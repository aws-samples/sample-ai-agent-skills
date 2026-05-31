# AWS CloudShell Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any CloudShell issue.

## Guardrail 1: Persistent Storage Is Only in $HOME
CloudShell provides 1 GB of persistent storage ONLY in the $HOME directory. All files outside $HOME (including /tmp, /var, /usr/local) are ephemeral and lost when the session ends. Do not assume installed packages or system-level changes persist.

## Guardrail 2: Sessions Time Out After 20 Minutes of Inactivity
CloudShell sessions automatically disconnect after 20 minutes of inactivity. This is not configurable. Long-running processes should use screen, tmux, or nohup. The session itself terminates after 12 hours maximum regardless of activity.

## Guardrail 3: VPC Environments Have No Internet by Default
VPC-connected CloudShell environments do NOT have internet access unless the subnet has a NAT gateway or VPC endpoints configured. Do not assume internet connectivity in VPC mode. Public subnets with an internet gateway do NOT provide internet to CloudShell VPC environments.

## Guardrail 4: CloudShell Inherits Console User Permissions
CloudShell uses the IAM credentials of the signed-in console user. There are no separate CloudShell IAM roles or credentials. Permission issues are IAM user/role issues, not CloudShell-specific.

## Guardrail 5: Regional Availability Is Limited
CloudShell is NOT available in all AWS regions. Before troubleshooting launch failures, verify the region supports CloudShell. Attempting to use CloudShell in an unsupported region produces confusing errors.

## Guardrail 6: File Transfer Limits Apply
File upload and download through the CloudShell web interface is limited to 1 GB per file. For larger files, use AWS CLI with S3 as an intermediary. The web upload/download is separate from CLI-based file operations.

## Guardrail 7: Concurrent Environment Limits Exist
Each IAM user can have a maximum of 10 concurrent CloudShell environments across all regions. Hitting this limit prevents new environment creation. Old environments must be deleted to free capacity.

## Guardrail 8: Pre-installed Tools May Be Outdated
CloudShell comes with pre-installed tools (AWS CLI, Python, Node.js, etc.) but versions may not be the latest. Custom tool installations in $HOME persist, but system-level installations via yum/apt do not survive session restarts.

## Guardrail 9: Network Restrictions Can Block CloudShell
Organizations using SCPs, VPC endpoint policies, or network firewalls can block CloudShell access. The CloudShell service requires specific endpoints. Corporate proxies may also interfere with WebSocket connections.

## Guardrail 10: Environment Deletion Is Permanent
Deleting a CloudShell environment permanently removes all persistent storage ($HOME contents) for that environment. This cannot be undone. Always back up important files to S3 before deleting environments.

## Guardrail 11: CloudShell Is Not a Compute Service
CloudShell is designed for CLI tasks and light scripting, not for running production workloads, servers, or resource-intensive computations. Resource limits (CPU, memory) are enforced and not configurable.

## Guardrail 12: Safe Paste Is Enabled by Default
CloudShell has a safe paste feature that warns when pasting multi-line commands. This is a security feature to prevent accidental execution of malicious clipboard content. It can be disabled per session but re-enables on new sessions.
