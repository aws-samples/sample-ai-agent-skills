# Bottlerocket Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Bottlerocket issue.

## Guardrail 1: No Package Manager — API-Driven Configuration Only
Bottlerocket has NO yum, apt, dnf, or any package manager. All configuration is done through the settings API (`apiclient`). Never suggest installing packages or editing configuration files directly on the host filesystem.

## Guardrail 2: ECS and EKS Variants Are Separate AMIs
Bottlerocket publishes separate AMI variants for ECS (`bottlerocket-aws-ecs-*`) and EKS (`bottlerocket-aws-k8s-*`). Using the wrong variant causes agent failures. Always verify the AMI variant matches the orchestrator.

## Guardrail 3: Admin Container Is Disabled by Default
The admin container provides shell access via `sheltie` but is disabled by default for security. It must be explicitly enabled via user data or SSM. Do not assume shell access is available without verifying admin container status.

## Guardrail 4: Control Container Provides SSM Access
The control container runs the SSM agent and is enabled by default. It provides `apiclient` and `logdog` for diagnostics. If SSM connectivity fails, the control container may be misconfigured or the instance may lack IAM permissions for SSM.

## Guardrail 5: Dual-Partition Update Scheme (A/B)
Bottlerocket uses A/B partitions for updates. The inactive partition is updated, then the system reboots. If an update fails, the system can roll back to the previous partition. Never assume an update is applied without confirming a reboot occurred.

## Guardrail 6: SELinux Is Enforced by Default
Bottlerocket enforces SELinux in enforcing mode. Custom container workloads that require host filesystem access or privileged operations may fail silently due to SELinux denials. Check audit logs before blaming application code.

## Guardrail 7: User Data Format Is TOML Not Shell Script
Bottlerocket user data uses TOML format for settings configuration, NOT shell scripts. Passing bash scripts as user data will be silently ignored. Always use TOML-formatted settings in user data.

## Guardrail 8: Bottlerocket Has No SSH by Default
There is no SSH daemon running by default. Access is through the admin container (if enabled) or the control container via SSM. Do not suggest SSH access without first enabling the admin container.

## Guardrail 9: Filesystem Is Read-Only
The root filesystem is read-only and dm-verity protected. Only specific paths (`/var`, `/opt`, `/etc` overlay) are writable. Attempts to modify the root filesystem will fail. Container data lives on a separate data partition.

## Guardrail 10: Settings Changes Require Commit and Reboot
Settings changes via `apiclient set` are staged. Some settings require `apiclient reboot` to take effect. Do not assume a setting is active immediately after `apiclient set` — verify with `apiclient get` after reboot if required.

## Guardrail 11: Log Collection Uses logdog Not Traditional Paths
Bottlerocket logs are collected using the `logdog` command in the control container, which bundles system logs, kernel logs, and container runtime logs. Traditional paths like `/var/log/messages` or `journalctl` are not directly accessible without the admin container.

## Guardrail 12: Network Configuration Is Settings-API Driven
Network settings (DNS, proxy, hostname) are configured through the settings API, not through `/etc/resolv.conf` or `/etc/hosts` directly. Manual edits to network files will be overwritten. Use `apiclient set` for persistent network configuration.
