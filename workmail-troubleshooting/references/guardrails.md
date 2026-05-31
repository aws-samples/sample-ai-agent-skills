# Amazon WorkMail Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any WorkMail issue.

## Guardrail 1: Organizations Are Regional
Each WorkMail organization is tied to a specific AWS region. Mailboxes, domains, and configuration are regional. Users must connect to the correct regional endpoint. There is no cross-region replication.

## Guardrail 2: Directory Deletion Destroys Everything
WorkMail organizations depend on a Directory Service directory. Deleting the directory permanently destroys the organization, all mailboxes, and all email content. Never delete a directory associated with an active WorkMail organization.

## Guardrail 3: Domain Verification Requires DNS Records
Custom domains require MX records pointing to WorkMail, TXT records for ownership verification, and CNAME records for DKIM signing. All three record types must be correctly configured. Partial DNS setup causes intermittent delivery issues.

## Guardrail 4: SES Underlies Outbound Email
WorkMail uses Amazon SES for outbound email delivery. SES sending limits, reputation, and suppression lists affect WorkMail delivery. If outbound email fails, check SES status and sending quotas in addition to WorkMail configuration.

## Guardrail 5: EAS Protocol for Mobile Devices
WorkMail mobile device sync uses Exchange ActiveSync (EAS) protocol, not IMAP or POP. Mobile device configuration must use EAS settings. IMAP is available for desktop clients but not recommended for mobile.

## Guardrail 6: Access Control Rules Are Evaluated in Order
Access control rules are evaluated in order. The first matching rule determines access. A deny rule before an allow rule blocks access even if a later rule would allow it. Always review rule order when troubleshooting access issues.

## Guardrail 7: Migration Tool Is Required for Exchange
Migrating from Microsoft Exchange requires the Amazon WorkMail Migration Tool. Direct mailbox import from PST, OST, or EDB files is not supported. The migration tool connects to the source Exchange server via EWS.

## Guardrail 8: Mailbox Size Limits Apply
Each user mailbox has a size limit (50 GB by default). Exceeding the limit blocks incoming email. Users must delete or archive email to free space. Mailbox size cannot be increased beyond the service limit.

## Guardrail 9: Group Email Has Delivery Limits
Distribution groups have member limits and message size limits. Large groups may experience delivery delays. Groups do not have their own mailbox — email is delivered to each member's mailbox individually.

## Guardrail 10: Journaling Captures All Email
Journaling rules capture copies of all email (sent and received) for compliance. Journal email is delivered to a specified address. Journaling increases storage usage and may have cost implications.

## Guardrail 11: Email Flow Rules Affect Delivery
Email flow rules (inbound and outbound) can modify, redirect, or block email. Misconfigured flow rules are a common cause of delivery failures. Always check flow rules when email is missing or misdirected.

## Guardrail 12: Retention Policies Affect Mailbox Content
Retention policies automatically delete email after a specified period. Users may report missing email that was deleted by retention policy. Check retention settings before investigating missing email as a delivery failure.
