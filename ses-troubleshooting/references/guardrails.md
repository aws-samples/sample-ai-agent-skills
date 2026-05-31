# SES Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any SES issue.

## Guardrail 1: Sandbox Mode Is Not a Bug
New SES accounts start in sandbox mode. In sandbox, you can only send to verified email addresses and domains. Daily sending quota is 200 emails. This is expected behavior, not a service issue. Production access must be explicitly requested.

## Guardrail 2: Bounce Rate Threshold Is 5%
AWS monitors bounce rate and will place the account under review if it exceeds 5%. Sending may be paused if the rate stays high. The bounce rate is calculated over a rolling period. Never cite a different threshold. Always check the current rate before diagnosing deliverability issues.

## Guardrail 3: Complaint Rate Threshold Is 0.1%
AWS monitors complaint rate (feedback loop reports from ISPs) and will review the account if it exceeds 0.1%. This is a much stricter threshold than bounce rate. Even a small number of complaints on low volume can trigger review.

## Guardrail 4: SMTP Credentials Are Not IAM Credentials
SES SMTP credentials are derived from IAM access keys using a specific signing algorithm. You cannot use IAM access key ID and secret access key directly for SMTP authentication. The SMTP password must be generated using the SES-specific algorithm or through the SES console.

## Guardrail 5: DKIM Requires All Three CNAME Records
SES Easy DKIM generates three CNAME records that must all be added to DNS. DKIM verification fails if any of the three records is missing or incorrect. Partial DKIM setup (1 or 2 records) does not provide partial signing — it fails completely.

## Guardrail 6: SPF Alone Does Not Align Your Domain
Default SES sending uses amazonses.com as the MAIL FROM domain. SPF passes for amazonses.com, not your domain. For SPF alignment (required for DMARC), you must configure a custom MAIL FROM domain and add the corresponding SPF record.

## Guardrail 7: Receipt Rules Process in Order
Receipt rules within a rule set are evaluated in order. The first rule whose recipient condition matches handles the email. Subsequent rules are NOT evaluated for that recipient. Rule ordering matters significantly for email routing.

## Guardrail 8: Suppression List Auto-Adds Addresses
The account-level suppression list automatically adds email addresses that generate hard bounces or complaints. Manually removing an address from the suppression list does not prevent it from being re-added if the next send also bounces. Fix the underlying deliverability issue first.

## Guardrail 9: Sending Quota and Rate Are Separate Limits
SES enforces both a daily sending quota (total emails per 24-hour period) and a per-second sending rate. Exceeding either limit results in throttling. The daily quota resets on a rolling 24-hour basis, not at midnight.

## Guardrail 10: Configuration Set Is Optional but Recommended
Emails can be sent without a configuration set. However, without one, you lose event tracking (delivery, bounce, complaint, open, click). Never claim a configuration set is required for basic sending.

## Guardrail 11: SES Receiving Is Region-Limited
SES email receiving is only available in specific AWS regions (us-east-1, us-west-2, eu-west-1). Not all SES regions support receiving. Never suggest setting up receipt rules in a region that doesn't support receiving.

## Guardrail 12: Verified Identity Required for From Address
Every From address must belong to a verified identity (email address or domain). Sending from an unverified address fails with MessageRejected error. Domain verification covers all addresses under that domain. Email verification covers only that specific address.
