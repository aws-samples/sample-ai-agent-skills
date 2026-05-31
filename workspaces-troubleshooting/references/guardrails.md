# WorkSpaces Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any WorkSpaces issue.

## Guardrail 1: Two Subnets in Different AZs Required
WorkSpaces directory registration requires at least two subnets in different Availability Zones. Even if you only launch WorkSpaces in one AZ, both subnets must exist and have available IP addresses. Never suggest single-subnet configurations.

## Guardrail 2: UNHEALTHY State May Be Transient
WorkSpaces health checks run every 20 minutes. A WorkSpace showing UNHEALTHY may recover on the next check cycle. Do not immediately escalate or rebuild based on a single UNHEALTHY status. Wait for at least two consecutive health check failures before diagnosing as a persistent issue.

## Guardrail 3: PCoIP and WSP Use Different Ports
PCoIP uses port 4172 (TCP and UDP). WSP uses port 4195 (TCP and UDP). Both protocols also require HTTPS (443) for authentication and registration. Misidentifying the protocol leads to opening wrong ports. Always verify which protocol the WorkSpace is configured to use before checking firewall rules.

## Guardrail 4: Web Access Requires WSP Protocol
WorkSpaces Web Access (browser-based) only works with the WSP streaming protocol. PCoIP does not support Web Access. Never suggest Web Access as an option for PCoIP-configured WorkSpaces. The WorkSpace must be migrated to WSP first.

## Guardrail 5: Rebuild Restores from Last Snapshot Not Real-Time
Rebuilding a WorkSpace restores the root volume from the latest bundle image and the user volume (D: drive) from the most recent automatic snapshot. Data created after the last snapshot is lost. Snapshots are taken every 12 hours. Never claim rebuild preserves all current user data.

## Guardrail 6: Custom Bundle Volume Sizes Cannot Be Reduced
Custom bundles inherit volume sizes from the source WorkSpace. Once created, root and user volume sizes in a bundle cannot be reduced, only increased. Never suggest reducing volume sizes in existing bundles. A new bundle must be created from a WorkSpace with smaller volumes.

## Guardrail 7: IP Access Control Groups Are Allowlists
IP access control groups define which client IP addresses can connect. They are allowlists, not blocklists. If an IP access control group is associated with a directory but contains no rules, all connections are blocked. An empty group is not the same as no group.

## Guardrail 8: AD Connector Requires Line of Sight to Domain Controllers
AD Connector needs network connectivity to on-premises domain controllers on ports 53 (DNS), 88 (Kerberos), 389 (LDAP), and 445 (SMB). All four ports must be open. Partial connectivity causes intermittent authentication failures that are difficult to diagnose.

## Guardrail 9: MFA Configuration Depends on Directory Type
AD Connector supports RADIUS-based MFA only. AWS Managed Microsoft AD supports both RADIUS and smart card authentication. Simple AD does not support MFA. Never suggest MFA configurations incompatible with the directory type.

## Guardrail 10: Auto-Stop WorkSpaces Have Reconnection Delay
Auto-stop WorkSpaces stop after the configured idle timeout. When a user reconnects, the WorkSpace must start up, which takes 1-3 minutes. This is expected behavior, not a connectivity issue. Never diagnose auto-stop startup time as a connection failure.

## Guardrail 11: WorkSpaces Streaming Gateway Requires Specific DNS
The WorkSpaces client resolves streaming gateway endpoints via DNS. If custom DNS servers are configured in the directory, they must be able to resolve AWS public DNS names. DNS resolution failures cause "unable to connect" errors that look like network issues.

## Guardrail 12: Maintenance Windows Affect Availability
WorkSpaces in AlwaysOn mode have a configurable maintenance window (default: Sunday 00:00-04:00 UTC). During maintenance, WorkSpaces may be rebooted for updates. WorkSpaces in ADMIN_MAINTENANCE state are inaccessible to users. Never diagnose maintenance-window unavailability as an outage.
