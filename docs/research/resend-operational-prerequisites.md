# Resend Operational Prerequisites

**Research ticket:** [#6](https://github.com/mattias-wiberg/saas-boilerplate/issues/6)
**Research date:** 2026-09-02
**Status:** Decision ready
**Source policy:** Current official Resend documentation only.

## Scope

This note covers the provider-side prerequisites and operating checklist for a
SaaS starter: account and team setup, API credentials, sender prerequisites,
safe testing, delivery signals, webhooks, smoke checks, failure visibility, and
credential lifecycle.

Application email feature contracts, message content, templates, recipient
policy, retry/user-notification behavior, and tenant/domain behavior are out of
scope. The unavoidable sender decision is only that production sending needs a
verified domain owned by the Resend account. This note does not choose a root
domain, subdomain, sender address, or per-tenant domain model. Resend recommends
subdomains for reputation separation, but that remains a later product or
operations decision. [Verified domains](https://resend.com/docs/dashboard/domains/introduction)

## Executive Finding

Resend is operationally ready for a starter once these gates are satisfied:

1. A Resend team is owned by the operator, with MFA enabled for human access.
2. A sending domain owned by the operator is added and verified.
3. The server has a least-privilege, sending-only API key stored as a deployment
   secret.
4. A public HTTPS webhook endpoint is registered if delivery-state visibility
   is required; its signing secret is stored server-side and requests are
   verified.
5. A provider-level smoke check confirms API acceptance, delivery events, safe
   failure simulations, dashboard/log visibility, and webhook receipt.

Resend says accounts have production access immediately and do not require a
separate production approval process. Domain verification and the API key are
still prerequisites for normal production sending. [Production approval](https://resend.com/docs/knowledge-base/does-resend-require-production-approval)
 [Introduction](https://resend.com/docs/introduction)

## Account And API Key Setup

- Sign up for the intended Resend account and work in an intentional team. A
  Resend account is the login, while a team is the environment holding API
  keys, domains, and email data; teams also have separate billing and usage.
  [Teams](https://resend.com/docs/dashboard/settings/team)
  [Account deletion](https://resend.com/docs/knowledge-base/how-can-i-delete-my-resend-account)
- Enable MFA on the human Resend account before provisioning production
  credentials. Resend documents MFA as an account setting that requires an
  authenticator code at sign-in. [MFA](https://resend.com/docs/knowledge-base/how-can-i-add-mfa)
- Code-based use requires at least one API key. Create it in the Dashboard, API,
  CLI, or MCP server; the key value is shown only once. [Create an API key](https://resend.com/docs/create-an-api-key)
- Prefer `sending_access` for an application sender. It can send email only;
  `full_access` can create, read, update, and delete any API resource. A
  sending-only key can optionally be restricted to one verified domain.
  [Create API key API reference](https://resend.com/docs/api-reference/api-keys/create-api-key)
- Give each deployed service its own named key so the Dashboard's last-used
  indicator and per-key logs can identify activity and a compromised key can be
  removed independently. [Manage API keys](https://resend.com/docs/dashboard/api-keys/introduction)
  [API key security](https://resend.com/docs/knowledge-base/how-to-handle-api-keys)
- Keep the API key server-side in an environment or deployment secret. Resend
  explicitly recommends environment variables, never committing or hard-coding
  keys, and never exposing them in browser or other client-side code.
  [Create an API key](https://resend.com/docs/create-an-api-key)
  [API key security](https://resend.com/docs/knowledge-base/how-to-handle-api-keys)
- If using the REST API directly, use the HTTPS-only base URL
  `https://api.resend.com`, send `Authorization: Bearer <key>`, and include a
  `User-Agent`; Resend says requests without `User-Agent` are rejected with
  `403`. Official SDKs and the CLI add required headers automatically.
  [API concepts](https://resend.com/docs/api-reference/introduction)

## Sender And Testing Prerequisites

- Resend sends using a domain the account owns. Add the domain and publish the
  exact DNS records Resend provides, including the documented DKIM and SPF
  records, before treating the sender as production-ready. Verification often
  completes within 15 minutes but can take up to 72 hours to propagate.
  [Add and verify a domain](https://resend.com/docs/add-a-domain)
- Once a domain is verified, Resend says any address at that domain can be used
  without separately creating an email address, sender identity, or from-address.
  No sender-address selection is made by this note. [Sending emails](https://resend.com/docs/dashboard/emails/introduction)
- The `resend.dev` domain is a testing path only and can send only to the email
  address associated with the Resend account. Sending to other recipients
  requires a verified owned domain and a `from` address on that domain.
  [resend.dev restriction](https://resend.com/docs/knowledge-base/403-error-resend-dev-domain)
- For controlled event tests, Resend documents these safe recipients:
  `delivered@resend.dev`, `bounced@resend.dev`,
  `complained@resend.dev`, and `suppressed@resend.dev`. Labels after `+` are
  supported for the first three, but not currently for `suppressed`.
  [Testing addresses](https://resend.com/docs/knowledge-base/what-email-addresses-to-use-for-testing)
- Do not test with fake addresses or a fake SMTP server. Resend says its test
  addresses simulate delivery, bounce, complaint, and suppression scenarios;
  test messages count against the account sending quota.
  [Send test emails](https://resend.com/docs/dashboard/emails/send-test-emails)

## Delivery Signals And Webhooks

The send API response identifies the email, but acceptance is not delivery.
Resend defines `email.sent` as a successful API request for which it will
attempt delivery, while `email.delivered` means successful delivery to the
recipient's mail server. The latter is not a claim that the message is visible
in the recipient's inbox. [email.sent](https://resend.com/docs/webhooks/emails/sent)
 [email.delivered](https://resend.com/docs/webhooks/emails/delivered)

For operational visibility, the useful email events are:

| Event | Provider meaning | Operational use |
| --- | --- | --- |
| `email.sent` | The API request succeeded and delivery will be attempted. | Correlate the provider acceptance with the returned email ID. [Source](https://resend.com/docs/webhooks/emails/sent) |
| `email.delivered` | Resend delivered the email to the recipient's mail server. | Record a delivery signal distinct from API acceptance. [Source](https://resend.com/docs/webhooks/emails/delivered) |
| `email.delivery_delayed` | Delivery could not complete because of a temporary issue. | Alert or inspect transient delivery problems. [Source](https://resend.com/docs/webhooks/emails/delivery-delayed) |
| `email.bounced` | The recipient's mail server permanently rejected the email. | Surface a permanent delivery failure. [Source](https://resend.com/docs/webhooks/emails/bounced) |
| `email.complained` | The email was delivered, then marked as spam. | Surface a sender-reputation signal. [Source](https://resend.com/docs/webhooks/emails/complained) |
| `email.failed` | The email failed because of an error; Resend lists invalid recipients, API key issues, domain verification, and quota limits as examples. | Surface provider-side send failures and their failure reason. [Source](https://resend.com/docs/webhooks/emails/failed) |
| `email.suppressed` | Resend suppressed the email. | Surface a provider suppression outcome. [Source](https://resend.com/docs/webhooks/emails/suppressed) |

### Webhook Requirements

- Register a publicly accessible HTTPS endpoint and select the event types to
  receive. Resend sends JSON over HTTPS; a successful receiver responds with
  HTTP `200`. Local development can use a public tunnel or the Resend CLI's
  webhook listener. [Managing webhooks](https://resend.com/docs/webhooks/introduction)
  [Create webhook API reference](https://resend.com/docs/api-reference/webhooks/create-webhook)
- Treat the webhook signing secret as a server-side credential. Resend exposes
  it on the webhook details page and in create, retrieve, and list responses.
  Verify the raw request body with the `svix-id`, `svix-timestamp`, and
  `svix-signature` headers before accepting the event. [Verify webhook requests](https://resend.com/docs/webhooks/verify-webhooks-requests)
- Assume at-least-once delivery and no ordering guarantee. Resend identifies
  each delivery with `svix-id`; a receiver must be able to recognize a
  duplicate and must not infer chronology from arrival order. [Managing webhooks](https://resend.com/docs/webhooks/introduction)
- The current retry guide lists attempts immediately, then after 5 seconds, 5
  minutes, 30 minutes, 2 hours, 5 hours, 10 hours, and another 10 hours. Resend
  emails the team when an endpoint starts failing and can automatically disable
  it; the endpoint can be re-enabled after recovery. [Retries and replays](https://resend.com/docs/webhooks/retries-and-replays)
- Resend supports replaying both failed and succeeded webhook messages from the
  Dashboard. This is an operational recovery tool, not a substitute for
  duplicate-safe receipt. [Retries and replays](https://resend.com/docs/webhooks/retries-and-replays)

The recommended minimum operational subscription is `email.sent`,
`email.delivered`, `email.delivery_delayed`, `email.bounced`,
`email.complained`, `email.failed`, and `email.suppressed`. Open/click and
domain/contact events are not prerequisites for provider readiness and are
deliberately not assigned application meaning here. The available event list is
documented by Resend. [Event types](https://resend.com/docs/webhooks/event-types)

## Failure Visibility

- The Emails Dashboard exposes email details and associated events; each email
  can link to the API request log that created it. [View and manage emails](https://resend.com/docs/dashboard/emails/manage-emails)
- The Logs Dashboard supports searching and filtering by status, date range,
  user agent, and API key. Log details include the endpoint, method, status,
  timestamp, full request body, and full response body. Treat this as sensitive
  operational data when deciding who can access it. [Logs](https://resend.com/docs/dashboard/logs/introduction)
- Resend uses `2xx` for success, `4xx` for request/account failures, and `5xx`
  for infrastructure failures. Important setup failures include missing or
  invalid credentials (`401`/`403`), domain restrictions (`403`), and rate or
  quota exhaustion (`429`). [API concepts](https://resend.com/docs/api-reference/introduction)
  [Errors](https://resend.com/docs/api-reference/errors)
- The default API rate limit is currently 10 requests per second per team,
  shared across the team's API keys. Responses expose rate-limit headers,
  including `ratelimit-remaining`, `ratelimit-reset`, and `retry-after`.
  [Usage limits](https://resend.com/docs/api-reference/rate-limit)
- Resend's current free-account documentation lists 100 transactional emails
  per day and 3,000 per month. Sent and received messages count, and multiple
  `To`, `CC`, or `BCC` recipients count separately. Check the account's current
  plan and Usage page rather than treating these figures as a permanent
  contract. [Account quotas and limits](https://resend.com/docs/knowledge-base/account-quotas-and-limits)
- Resend says accounts must stay below a 4% bounce rate and 0.08% spam rate;
  exceeding either may temporarily pause sending. Monitor the Metrics page and
  the corresponding bounce/complaint events. [Account quotas and limits](https://resend.com/docs/knowledge-base/account-quotas-and-limits)
- Resend retains email data, delivery status/events, logs, and metrics for 30
  days across Free, Pro, and Scale plans. If an operational or compliance need
  exceeds that window, retain the required webhook evidence in an authorized
  system with its own access and retention controls. [Account quotas and limits](https://resend.com/docs/knowledge-base/account-quotas-and-limits)

## Provider-Level Smoke Check

Run this check after provisioning and repeat it after changing the API key,
verified domain, webhook endpoint, or deployment secret. It intentionally tests
provider operations only; it does not assert an application email contract.

1. Confirm the intended team, MFA, verified domain status, key name/scope, and
   deployment secret. If using raw HTTP, confirm HTTPS, bearer authentication,
   and `User-Agent`.
2. Send one small test message to `delivered@resend.dev` (or, on the limited
   unverified `resend.dev` path, to the account's own email address). Capture the
   returned email ID and HTTP result. A successful API response proves request
   acceptance only.
3. In the Emails Dashboard, confirm the email record and event state. In Logs,
   filter by the key and inspect the matching request/response and status.
4. If a webhook is configured, verify the signed raw payload, correlate its
   email ID, confirm the endpoint returned `200`, and confirm duplicate delivery
   would be recognized by `svix-id`.
5. Exercise the safe `bounced`, `complained`, and `suppressed` test recipients
   and confirm the corresponding failure/suppression signals are visible. Do
   not use fake recipient addresses; remember that test sends consume quota.
6. Confirm the Usage/Metrics view and, where relevant, the rate-limit headers
   are understood by the operator. Stop the release if the domain is pending,
   the key is exposed client-side, the webhook cannot be verified, or the
   expected Dashboard/Log evidence is absent.

The test addresses, event simulations, Dashboard views, log fields, and
signature requirements above are all documented by Resend. [Send test emails](https://resend.com/docs/dashboard/emails/send-test-emails)
 [Logs](https://resend.com/docs/dashboard/logs/introduction)
 [Verify webhook requests](https://resend.com/docs/webhooks/verify-webhooks-requests)

## Operational Lifecycle

### Provision

Create the team, enable human MFA, add and verify the owned sending domain,
create a named sending-only key with a domain restriction where appropriate,
store the key and webhook secret server-side, and register the HTTPS webhook.
Do not treat a pending DNS verification as production-ready. [Create an API key](https://resend.com/docs/create-an-api-key)
 [Add and verify a domain](https://resend.com/docs/add-a-domain)
 [Verify webhook requests](https://resend.com/docs/webhooks/verify-webhooks-requests)

### Release And Operate

Run the smoke check, then watch the Emails Dashboard, Logs, Metrics, Usage
limits, and webhook failure notifications. Keep API acceptance, delayed
delivery, permanent bounce, complaint, suppression, and provider failure as
distinct operational signals. [View and manage emails](https://resend.com/docs/dashboard/emails/manage-emails)
 [Logs](https://resend.com/docs/dashboard/logs/introduction)
 [Retries and replays](https://resend.com/docs/webhooks/retries-and-replays)

### Rotate

Resend keys do not expire automatically. Its documented rotation sequence is:
create a replacement with the same permission and domain scope, deploy it
everywhere, verify recent requests by filtering Logs by the new key, then delete
the old key. Resend recommends rotation at least every 90 days and flags keys
unused for 30 or more days for review. [API key security](https://resend.com/docs/knowledge-base/how-to-handle-api-keys)

### Respond To A Leak

Delete a leaked key immediately; Resend says deletion revokes it without a
separate revoke step. Create and deploy a replacement, then review the Logs and
Emails pages for the affected key and time window and watch Metrics for
reputation impact. [Leaked API keys](https://resend.com/docs/knowledge-base/how-to-handle-a-leaked-api-key)

### Retire

When a sender is retired, remove its deployment secret, delete its unused API
key, and disable or remove its webhook. Resend stops webhook delivery attempts
when an endpoint is removed or disabled. Full account/team deletion is a
separate, irreversible administrative action. [Retries and replays](https://resend.com/docs/webhooks/retries-and-replays)
 [Manage API keys](https://resend.com/docs/dashboard/api-keys/introduction)
 [Account deletion](https://resend.com/docs/knowledge-base/how-can-i-delete-my-resend-account)

## Recommended Decision Set

1. Require a verified owned domain for production sending; defer root versus
   subdomain and all sender/domain behavior decisions.
2. Operate one intentional Resend team with MFA for humans, and use a distinct
   sending-only, optionally domain-restricted key per deployed service.
3. Keep API keys and webhook signing secrets server-side; never put them in
   client code, source control, or public logs.
4. Register one HTTPS webhook for the minimum operational email events, verify
   signatures over the raw body, and make receipt duplicate-safe and tolerant
   of out-of-order delivery.
5. Make the smoke check and Dashboard/Logs/Metrics review release gates; treat
   API success as acceptance, not proof of delivery.
6. Rotate keys at least every 90 days, review unused keys after 30 days, and
   delete a suspected leak immediately.

These decisions establish provider readiness without defining application email
features or tenant/domain behavior.

## Sources

- [Resend introduction](https://resend.com/docs/introduction)
- [Create an API key](https://resend.com/docs/create-an-api-key)
- [Manage API keys](https://resend.com/docs/dashboard/api-keys/introduction)
- [API key API reference](https://resend.com/docs/api-reference/api-keys/create-api-key)
- [API key security](https://resend.com/docs/knowledge-base/how-to-handle-api-keys)
- [Leaked API keys](https://resend.com/docs/knowledge-base/how-to-handle-a-leaked-api-key)
- [Teams](https://resend.com/docs/dashboard/settings/team)
- [MFA](https://resend.com/docs/knowledge-base/how-can-i-add-mfa)
- [Production approval](https://resend.com/docs/knowledge-base/does-resend-require-production-approval)
- [Add and verify a domain](https://resend.com/docs/add-a-domain)
- [Verified domains](https://resend.com/docs/dashboard/domains/introduction)
- [Sending emails](https://resend.com/docs/dashboard/emails/introduction)
- [resend.dev restriction](https://resend.com/docs/knowledge-base/403-error-resend-dev-domain)
- [Testing addresses](https://resend.com/docs/knowledge-base/what-email-addresses-to-use-for-testing)
- [Send test emails](https://resend.com/docs/dashboard/emails/send-test-emails)
- [API concepts](https://resend.com/docs/api-reference/introduction)
- [Errors](https://resend.com/docs/api-reference/errors)
- [Usage limits](https://resend.com/docs/api-reference/rate-limit)
- [Account quotas and limits](https://resend.com/docs/knowledge-base/account-quotas-and-limits)
- [View and manage emails](https://resend.com/docs/dashboard/emails/manage-emails)
- [Logs](https://resend.com/docs/dashboard/logs/introduction)
- [Event types](https://resend.com/docs/webhooks/event-types)
- [email.sent](https://resend.com/docs/webhooks/emails/sent)
- [email.delivered](https://resend.com/docs/webhooks/emails/delivered)
- [email.delivery_delayed](https://resend.com/docs/webhooks/emails/delivery-delayed)
- [email.bounced](https://resend.com/docs/webhooks/emails/bounced)
- [email.complained](https://resend.com/docs/webhooks/emails/complained)
- [email.failed](https://resend.com/docs/webhooks/emails/failed)
- [email.suppressed](https://resend.com/docs/webhooks/emails/suppressed)
- [Managing webhooks](https://resend.com/docs/webhooks/introduction)
- [Create webhook API reference](https://resend.com/docs/api-reference/webhooks/create-webhook)
- [Verify webhook requests](https://resend.com/docs/webhooks/verify-webhooks-requests)
- [Retries and replays](https://resend.com/docs/webhooks/retries-and-replays)
- [Account deletion](https://resend.com/docs/knowledge-base/how-can-i-delete-my-resend-account)
