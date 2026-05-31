---
name: workspaces-web-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon WorkSpaces Web
  (WorkSpaces Secure Browser) problems by analyzing portal creation,
  configuration, browser sessions, connectivity, identity provider
  setup, SAML authentication, network settings, VPC configuration,
  browser policies, user access logging, IP access settings, and
  following structured runbooks.
  Activate when: portal creation failures, browser session errors,
  IdP configuration issues, SAML authentication failures, network
  problems, browser policy issues, user access logging gaps, IP
  access restrictions, or the user says something is wrong with
  WorkSpaces Web.
compatibility: >
  Requires AWS CLI or SDK access with workspaces-web:*,
  ec2:Describe*, iam:Get*, kms:Describe*, logs:*, and
  cloudwatch:* permissions. Identity provider admin access may
  be required for SAML troubleshooting.
---

# Amazon WorkSpaces Web Diagnostics

## When to use

Any Amazon WorkSpaces Web investigation — portal creation, configuration, browser sessions, connectivity, identity provider setup, SAML authentication, network settings, VPC configuration, browser policies, user access logging, or IP access settings.

## Investigation workflow

### Step 1 — Collect and triage

```
aws workspaces-web list-portals --query 'portals[*].{PortalArn:portalArn,PortalEndpoint:portalEndpoint,PortalStatus:portalStatus,DisplayName:displayName}'
aws workspaces-web list-browser-settings --query 'browserSettings[*].{BrowserSettingsArn:browserSettingsArn}'
aws workspaces-web list-network-settings --query 'networkSettings[*].{NetworkSettingsArn:networkSettingsArn}'
```

### Step 2 — Domain deep dive

```
aws workspaces-web get-portal --portal-arn <portal-arn>
aws workspaces-web get-browser-settings --browser-settings-arn <arn>
aws workspaces-web get-identity-provider --portal-arn <portal-arn> --identity-provider-arn <idp-arn>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=workspaces-web.amazonaws.com --max-results 20
aws workspaces-web get-network-settings --network-settings-arn <arn>
aws workspaces-web get-user-access-logging-settings --user-access-logging-settings-arn <arn>
```

Read `references/guardrails.md` before concluding on any WorkSpaces Web issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `workspaces-web list-portals` | List all web portals |
| `workspaces-web get-portal` | Get portal details and status |
| `workspaces-web get-browser-settings` | Get browser policy configuration |
| `workspaces-web get-network-settings` | Get VPC and network configuration |
| `workspaces-web get-identity-provider` | Get IdP/SAML configuration |
| `workspaces-web get-user-access-logging-settings` | Get logging configuration |
| `workspaces-web get-ip-access-settings` | Get IP restriction rules |

## Gotchas: Amazon WorkSpaces Web

- WorkSpaces Web provides managed browser-based access to internal websites and SaaS applications. It is NOT a full desktop — it is a browser-only experience.
- Portals are the primary resource. Each portal has its own endpoint URL, browser settings, network settings, and identity provider configuration.
- Browser sessions run in AWS-managed infrastructure. You do NOT manage EC2 instances, containers, or operating systems for the browser environment.
- SAML 2.0 is the only supported authentication method. WorkSpaces Web does not support OIDC, basic auth, or IAM-only authentication for end users.
- Network settings define the VPC, subnets, and security groups for browser session traffic. Sessions access internal resources through this VPC configuration.
- Browser policies control clipboard, file transfer, printing, and other browser capabilities. Policies are defined as Chrome enterprise policy JSON.
- IP access settings restrict which client IP addresses can connect to the portal. This is client-side IP filtering, not server-side network ACLs.

## Anti-hallucination rules

1. Always cite specific portal ARNs, browser settings ARNs, or API responses as evidence.
2. WorkSpaces Web is browser-only, NOT a full desktop. Never suggest desktop-level troubleshooting.
3. SAML 2.0 is the only auth method. Never suggest OIDC or other authentication protocols.
4. Browser sessions are AWS-managed. Never suggest SSH into session hosts or OS-level debugging.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Portal | A1–A2 | Portal creation failures, portal configuration |
| B — Sessions | B1–B2 | Browser session errors, session connectivity |
| C — Identity | C1–C2 | Identity provider configuration, SAML authentication |
| D — Network | D1–D2 | Network settings, VPC configuration |
| E — Policy | E1–E2 | Browser policy issues, user access logging |
| F — Access | F1 | IP access settings |
| Z — Catch-All | Z1 | General troubleshooting |
