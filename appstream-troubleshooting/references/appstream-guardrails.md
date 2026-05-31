# Amazon AppStream 2.0 Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon AppStream 2.0 issue.

## Guardrail 1: Fleet Instances Are NOT Directly Manageable EC2 Instances
AppStream fleet instances are managed by the service. You cannot SSH into them, install software on running instances, or modify their configuration directly. All software installation and configuration must be done through image builders. Fleet instances are created from images and are immutable during their lifecycle.

## Guardrail 2: Image Builders Are for Image Creation Only
Image builders are temporary instances used to install applications and create images. They are NOT for production streaming. Connect to the image builder via the AppStream console, install and configure applications, then create an image. Delete the image builder after creating the image to avoid unnecessary costs.

## Guardrail 3: Always-On vs On-Demand Fleet Types
Always-on fleets keep instances running and ready for immediate session start (higher cost, faster user experience). On-demand fleets start instances when users connect (lower cost, 1-2 minute session start delay). Elastic fleets use a different model with app blocks. Choose based on user experience requirements and cost constraints.

## Guardrail 4: NICE DCV Is the Streaming Protocol
AppStream 2.0 uses NICE DCV for streaming, not RDP or VNC. NICE DCV requires HTTPS (port 443) and optionally UDP 8433 for enhanced streaming. Firewalls, proxies, and network security must allow these ports. WebSocket connections are used for the streaming session. Do not suggest RDP or other remote desktop protocols.

## Guardrail 5: User Pool and SAML Are Mutually Exclusive Per Stack
A stack uses either the AppStream user pool OR SAML federation for authentication, not both. User pool is built-in user management with email-based accounts. SAML integrates with external identity providers (Okta, Azure AD, ADFS). Changing authentication type requires stack reconfiguration.

## Guardrail 6: Home Folders Are S3-Backed
Home folders provide persistent storage for users across sessions. They are backed by S3 buckets created automatically by AppStream. Each user gets a folder in the S3 bucket. Home folder content persists across sessions. S3 storage costs apply. The bucket name follows the pattern: appstream2-<hash>-<region>.

## Guardrail 7: App Settings Persistence Uses S3
Application settings (browser bookmarks, IDE preferences, Windows settings) are saved to S3 and restored in new sessions. This must be explicitly enabled on the stack. Settings are stored per user. There is a size limit for app settings. Not all application settings are captured — only those in supported Windows locations.

## Guardrail 8: Fleet Auto-Scaling Uses CloudWatch Metrics
Fleet scaling policies use CloudWatch metrics like CapacityUtilization. Configure minimum capacity (always available), maximum capacity (cost limit), and desired capacity. Scaling cooldown periods prevent rapid oscillation. InsufficientCapacityError metric indicates users couldn't get sessions due to capacity limits.

## Guardrail 9: Session Timeouts Are Separate Settings
Idle disconnect timeout: disconnects users after inactivity (5-360 minutes). Disconnect timeout: preserves disconnected sessions for reconnection (1-360 minutes, or never). Max session duration: maximum total session length (up to 5760 minutes / 96 hours). These are independent settings on the fleet.

## Guardrail 10: VPC Configuration Is Required for Custom Networking
Fleets can run in a VPC for access to internal resources. Specify at least two subnets in different AZs. Security groups control network access. Without a VPC, fleets use AppStream-managed networking with internet access. VPC fleets need a NAT gateway or VPC endpoints for internet and AWS service access.

## Guardrail 11: Streaming Endpoints Require Network Access
Users connect to AppStream streaming endpoints via HTTPS. The streaming URL is generated when a session starts. Corporate firewalls must allow access to AppStream streaming endpoints (*.appstream2.<region>.aws.amazon.com). Proxy servers must support WebSocket connections. Network issues cause session failures or high latency.

## Guardrail 12: Image Lifecycle Affects Fleet Operations
Images are immutable once created. To update applications, create a new image from an image builder. Fleets reference a specific image. Updating a fleet's image requires stopping the fleet, changing the image, and restarting. You cannot delete an image that is in use by a fleet. Plan image updates during maintenance windows.
