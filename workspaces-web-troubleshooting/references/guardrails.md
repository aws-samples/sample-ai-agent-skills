# Amazon WorkSpaces Web Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any WorkSpaces Web issue.

## Guardrail 1: Browser-Only Experience Not Full Desktop
WorkSpaces Web provides a managed browser-only experience. It is NOT a full virtual desktop like Amazon WorkSpaces. Users get a Chrome-based browser session, not a Windows or Linux desktop. Do not suggest desktop-level troubleshooting or OS configuration.

## Guardrail 2: SAML 2.0 Is the Only Authentication Method
WorkSpaces Web exclusively uses SAML 2.0 for end-user authentication. It does not support OIDC, OAuth, basic authentication, or IAM-only authentication for portal access. All identity provider configuration must be SAML-based.

## Guardrail 3: Browser Sessions Are AWS-Managed
The browser session infrastructure is fully managed by AWS. There are no EC2 instances, containers, or operating systems to manage. You cannot SSH into session hosts, install software, or modify the browser runtime. All customization is through browser settings and policies.

## Guardrail 4: Portal Is the Central Resource
Each WorkSpaces Web portal is a self-contained unit with its own endpoint URL, browser settings, network settings, identity provider, and access policies. Changes to one portal do not affect others. Portal status must be ACTIVE for users to connect.

## Guardrail 5: Network Settings Control Session Egress
Network settings define the VPC, subnets, and security groups through which browser sessions access internal and external resources. This is the session's network path, not the client's network path. Misconfigured network settings cause sessions to fail to reach target websites.

## Guardrail 6: Browser Policies Use Chrome Enterprise Format
Browser policies are defined using Chrome enterprise policy JSON format. Not all Chrome policies are supported — only a subset relevant to the managed browser environment. Invalid policy JSON causes browser settings creation to fail.

## Guardrail 7: IP Access Settings Filter Client IPs
IP access settings restrict which client IP addresses can connect to the portal. This is evaluated at the portal endpoint, not at the VPC level. Clients behind NAT will be seen by their public IP. Misconfigured IP rules lock out legitimate users.

## Guardrail 8: Portal Endpoint Is Region-Specific
Each portal has a unique endpoint URL in the format https://<id>.workspaces-web.<region>.amazonworkspaces.com. The portal is regional — it cannot be accessed from a different region's endpoint. Users must use the correct portal URL.

## Guardrail 9: Trust Store Certificates for Internal Sites
To access internal websites with private TLS certificates, you must upload the CA certificates to a trust store and associate it with the portal. Without the trust store, browser sessions show certificate errors for internal sites.

## Guardrail 10: Session Duration Has Limits
Browser sessions have configurable idle and total session timeouts. Default idle timeout is 15 minutes. Sessions that exceed the total duration are terminated. Users lose unsaved work when sessions end.

## Guardrail 11: User Access Logging Requires Kinesis Data Stream
User access logging sends session activity data to an Amazon Kinesis Data Stream. The Kinesis stream must exist and the WorkSpaces Web service role must have permission to write to it. Logging is not enabled by default.

## Guardrail 12: Cross-Account VPC Access Requires Proper Setup
If browser sessions need to access resources in a different AWS account, the VPC network settings must include proper cross-account networking (VPC peering, Transit Gateway, or PrivateLink). WorkSpaces Web does not provide built-in cross-account access.
