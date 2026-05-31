# Cognito Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Cognito issue.

## Guardrail 1: User Pool ≠ Identity Pool
User pools handle authentication (sign-up, sign-in, tokens). Identity pools handle authorization (temporary AWS credentials via STS). They are separate services with separate configurations. Do not conflate them.

## Guardrail 2: Token Expiry Is Fixed for ID Tokens
ID tokens always expire after 1 hour and cannot be configured. Access tokens can be configured (5 min to 24 hours). Refresh tokens can be configured (1 hour to 3,650 days). Only refresh tokens can obtain new ID/access tokens without re-authentication.

## Guardrail 3: App Client Secret Incompatible with SPA/Mobile
App clients with a secret cannot be used in browser-based or mobile applications. JavaScript and mobile SDKs cannot securely store the secret. Always use app clients WITHOUT a secret for public clients.

## Guardrail 4: Lambda Triggers Have 5-Second Timeout
All Cognito Lambda triggers must complete within 5 seconds. This cannot be increased. Complex operations (external API calls, database lookups) must be optimized or moved to post-authentication async processing.

## Guardrail 5: Custom Auth Requires Three Triggers
Custom authentication flow requires ALL three Lambda triggers: Define Auth Challenge, Create Auth Challenge, and Verify Auth Challenge Response. Missing any one causes the entire flow to fail silently or with cryptic errors.

## Guardrail 6: SAML Attribute Mapping Is Case-Sensitive
SAML attribute names must match exactly between the IdP and Cognito configuration. A mismatch in case or naming causes silent mapping failures where users authenticate but attributes are empty.

## Guardrail 7: User Pool Deletion Is Permanent
Deleting a user pool permanently removes all users, groups, triggers, and configurations. There is no soft-delete, recycle bin, or recovery mechanism. Always export users before deletion.

## Guardrail 8: SMS MFA Requires SNS Sandbox Exit
In SNS sandbox mode, SMS messages are only sent to verified phone numbers. Production MFA requires requesting SNS sandbox exit. Without it, unverified users cannot receive MFA codes.

## Guardrail 9: Hosted UI Customization Limits
Hosted UI supports only CSS (max 200 KB) and logo (max 100 KB) customization. Custom HTML pages, JavaScript injection, or layout changes are not supported. For full UI control, build a custom UI using the Cognito API.

## Guardrail 10: Refresh Token Revocation Scope
Revoking a refresh token only invalidates that specific token and its descendants. Previously issued access and ID tokens remain valid until they expire. For immediate session termination, use GlobalSignOut.

## Guardrail 11: Identity Pool Role Resolution Order
Identity pool role selection follows: rules-based mapping → token-based mapping → default role. If rules don't match and no default is set, the user gets no credentials. Always configure a default authenticated role.

## Guardrail 12: Advanced Security Pricing
Advanced security features (adaptive authentication, compromised credentials detection) are billed per monthly active user on top of base Cognito pricing. They are not included in the free tier of 50,000 MAUs.
