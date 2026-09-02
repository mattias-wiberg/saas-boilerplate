# Polar Operational Prerequisites

**Research date:** 2026-09-02
**Status:** AFK research note for issue #5
**Scope:** Polar account and organization operations, catalog setup, API credentials, webhook/event configuration, environment separation, secret handling, and provider-level smoke/lifecycle checks for a SaaS starter. Application billing logic, subscription lifecycle behavior, entitlement modeling, and authorization decisions are out of scope.

## Recommended Decision Set

1. **Use two isolated Polar environments.** Create a dedicated sandbox user and organization for development, and keep production users, organizations, tokens, product IDs, webhook endpoints, and secrets separate. Polar documents sandbox as isolated from production and requires a sandbox-created access token. [S1][S2]
2. **Use organization access tokens (OATs) for server-side work.** Create one named, expiring, least-privileged token per environment and operational purpose. Do not put an OAT in browser code, public repositories, or logs. [S3][S4]
3. **Treat the Polar catalog as configuration, not application state.** Give each sellable offer a deliberate product ID, visibility, price/currency/tax configuration, and stable metadata key. Use a sandbox copy for tests and archive products instead of treating them as deletable records. [S5]
4. **Use one raw webhook endpoint per environment.** Generate or set an endpoint secret, subscribe only to the checkout/order/customer events needed by the current integration, validate signatures with Polar's SDK, and monitor delivery health. [S6][S7][S8]
5. **Make account readiness a production gate.** Before live paid traffic, confirm the organization is eligible, the owner has completed review/KYC, a working payout account is connected, and the public website and support details are ready. [S9][S10][S11]
6. **Prove the transport path in sandbox, then use a zero-value production canary.** Use Polar's test card only in sandbox. If production wiring must be checked, use a free product or a 100% discount rather than real card details. [S2][S9]

## Account And Organization

- Sign up for Polar and create an organization. Polar's quick start describes sign-up with GitHub, Google, or email, followed by an organization for products and customers. [S1]
- Record the production and sandbox organization IDs and slugs separately. The organization name appears in checkout, the customer portal, and emails; the slug appears in checkout, the customer portal, and credit-card statements, so use intentional values. [S21]
- Set an official website and public support email, and keep the live website reachable before requesting review. The organization API describes these fields as the official website and public support email; Polar's review guidance recommends a live website and a complete end-to-end setup. [S9][S21]
- Assign operational access through organization membership. Polar has exactly one owner; admins can manage members, finances, organization settings, webhooks, and API keys; members can handle day-to-day products, customers, orders, and analytics but not those settings. [S13]
- Check that the business or individual is in Polar's current payout-country list. Polar distinguishes global customer payments from the countries supported for Stripe Connect Express payouts. [S11]
- Treat the following as the live-account gate: submit business details, complete owner identity verification, and connect a Stripe Connect Express payout account. Polar says the first review can take up to 14 days, and its payout documentation calls a working payout account a prerequisite to accepting money. [S9][S10]
- For payout onboarding, select the country of residence for a personal account or the organization's country of tax residency for a business. Polar documents the connected bank account as needing to be in the business's country and local currency. [S10]
- Keep a named owner responsible for Polar support and risk requests. Polar says merchants support their own customers and expects a response within 48 hours when Polar includes them in a support thread. [S9]

## Products And Catalog

- Polar treats both one-time purchases and recurring offers as products. Product creation exposes visibility states (`draft`, `private`, and `public`), and the product's billing cycle and pricing type are locked at creation; changing either requires a new product. [S5][S14]
- For the starter, choose a small canonical catalog and record each environment's product ID, name, visibility, price type, currency, tax behavior, and metadata in the operational inventory. The dashboard exposes a product ID, and the Checkout API requires at least a Product ID. [S5][S15]
- Use product metadata for a stable external correlation key. Polar says metadata is not customer-visible and is carried to related orders and webhooks. [S5]
- Decide whether the integration uses a persistent Checkout Link or creates sessions through the Checkout API. A Checkout Link is long-lived; each visit creates a short-lived Checkout Session, so do not distribute a generated session URL as the permanent product link. [S15]
- For a dynamic integration, the Checkout API returns the customer-facing checkout URL from a Product ID. If multiple currencies are enabled, record the organization's default presentment currency and verify geolocation behavior; for a server-created session, Polar says to forward the customer's IP as `customer_ip_address` so the server's IP is not used for currency and country detection. [S5][S16]
- Archive rather than delete a retired product. Polar says archived products disappear from new checkouts and can be unarchived; it does not provide permanent product deletion in the dashboard flow. [S5]
- Keep subscription and benefit consequences out of this runbook. Product configuration is recorded here only so the catalog can be created, identified, tested, and retired safely.

## API Credentials

- Use an Organization Access Token (OAT), which Polar recommends for organization API access and binds to one organization. Create it from organization **Settings**, under **Developers**, with a descriptive name, expiration date, and only the required scopes. [S3][S4]
- Create separate sandbox and production OATs. Polar says a production token cannot be used in sandbox, requires a sandbox-created token for sandbox API access, and documents separate base URLs: `https://sandbox-api.polar.sh/v1` and `https://api.polar.sh/v1`. [S2][S17]
- Match scopes to the smoke path. The current API reference declares `products:read`/`products:write` for product listing, `checkouts:write` for creating a checkout session, and `webhooks:write` for creating or updating a webhook endpoint; webhook listing requires the documented webhook read/write permissions. [S18][S19][S20]
- Keep OATs server-side in the deployment secret store. Polar explicitly says bearer tokens must remain private and never be exposed in client-side code; its API overview also warns against public repositories and logs. [S3][S17]
- Record token owner, purpose, environment, scope set, and expiration without recording the token value in the repository. Replace an expired or exposed token and rerun the read-only API smoke check before enabling traffic. Polar reports that its secret-scanning integrations automatically revoke a detected Polar token leak, but this does not make a leaked token safe to keep configured. [S3][S17]
- Do not use an OAT in a browser. Polar documents customer access tokens as a separate, restricted customer-portal mechanism; that customer-facing flow is outside this operational-prerequisites note. [S17]

## Webhooks And Events

- Add the endpoint in the organization's webhook settings, provide its URL, leave the format as **Raw** for a normal custom integration, set or generate the signing secret, and select the event types. Polar's setup guide documents each of these steps. [S6]
- Store the webhook secret separately for sandbox and production, alongside the endpoint ID and environment. Polar signs webhook requests and its TypeScript and Python SDKs provide validation helpers; use those helpers rather than inventing a verifier for the starter. [S6][S7]
- If a custom verifier is unavoidable, follow the current encoding note in Polar's delivery documentation. Polar currently documents an HMAC-SHA256 and `whsec_` handling nuance and says its SDKs accommodate both the Polar key and the Standard Webhooks key; the exact note should be revisited when changing SDK versions. [S7]
- Recommended initial event selection:
  - **Checkout transport:** `checkout.created`, `checkout.updated`, and `checkout.expired`.
  - **Order/payment transport:** `order.created`, `order.paid`, `order.updated`, and `order.refunded`.
  - **Customer record audit:** `customer.created`, `customer.updated`, and `customer.deleted` when the starter needs customer synchronization.
  - **Catalog audit (optional):** `product.created` and `product.updated` when catalog changes need an audit trail.
  - **Explicitly omitted here:** subscription, benefit, and benefit-grant events, because their application behavior is outside this ticket. Polar's event guide and webhook API enumerate the available event families. [S6][S8][S20]
- Make the receiver publicly reachable without application authorization middleware blocking it, and configure the final URL rather than a redirect. Polar treats 3xx responses as failures and documents excluding webhook routes from authorization middleware. [S7]
- Acknowledge quickly. Polar currently times out after 10 seconds, recommends responding within 2 seconds, retries failed deliveries up to 10 times with exponential backoff, and disables an endpoint after 10 consecutive non-2xx deliveries. A disabled endpoint must be manually enabled again in webhook settings. [S7]
- Use the delivery overview as the operational record: Polar documents historic deliveries, payload inspection, and redelivery after failure. [S7]
- If a firewall or reverse proxy uses an IP allowlist, reconcile it with Polar's current production and sandbox webhook IP ranges immediately before go-live; Polar documents different environment ranges and warns when a production address changes. [S7]
- For local development, Polar documents `polar login` followed by `polar listen <local-url>`; the CLI selects an organization and supplies a webhook secret to place in the environment. [S12]

## Sandbox And Live Operation

- Use Polar's sandbox as a separate development server, not as a production test-mode flag. Polar says sandbox users, organizations, data, and tokens are isolated from production and that a production token cannot be used there. [S2][S17]
- Create a dedicated sandbox user and organization, then use the sandbox API base URL and sandbox OAT. The sandbox supports the complete customer funnel and Stripe test card numbers; Polar documents `4242 4242 4242 4242` with a future expiry and random CVC as the easiest successful-payment test. [S2]
- Expect sandbox email behavior to differ. Polar says customer-facing emails in sandbox are delivered only to organization members, while sub-addressing aliases are accepted. [S2]
- Never run real-card test purchases in production. Polar classifies that behavior as card testing and directs merchants to sandbox; for a production verification, it recommends a free product or a 100% discount code. [S9]
- Treat production as a separate configuration promotion, not a copy-and-switch operation. Reconfirm the production organization, product IDs, checkout URL, OAT, webhook endpoint, webhook secret, and support/website details before opening traffic. The separate IDs and credentials follow from Polar's isolated environments and organization-scoped resources. [S2][S4][S17][S21]

## Smoke And Lifecycle Checklist

### Sandbox smoke

1. **Identity:** Sign in to the sandbox, select the intended organization, and record its name, slug, ID, owner, and member roles. [S2][S12][S13]
2. **Credential:** Use only the sandbox OAT and sandbox base URL. List products and confirm a successful response is scoped to the intended organization; a failure is a credential/scope/environment issue, not a checkout issue. [S2][S17][S18]
3. **Catalog:** Confirm the test product's ID, visibility, price/currency, tax behavior, and external metadata. Confirm it is not archived before using it in checkout. [S5][S18]
4. **Checkout:** Use the persistent Checkout Link or create a Checkout Session with the test Product ID. Open the returned URL and complete one sandbox payment with the documented test card; record the resulting checkout ID and visible success state. [S2][S15][S16]
5. **Webhook delivery:** Confirm the selected checkout/order events appear in Polar's delivery history, the receiver validates the signature with the sandbox secret, and the endpoint remains enabled after a 2xx response. [S6][S7][S8]
6. **Negative path:** Send a request with the wrong secret or otherwise exercise signature failure and confirm the receiver rejects it. The local-webhook documentation specifically describes a 403 when the configured secret is wrong. [S7][S12]
7. **Recovery path:** In sandbox only, exercise a controlled delivery failure and confirm retry visibility, redelivery, endpoint disablement/re-enable behavior, and firewall or proxy logs where applicable. Do not use production to test failure handling. [S2][S7]

### Catalog and configuration lifecycle

1. Change a non-destructive product field in sandbox and confirm the product remains associated with the expected organization and the `product.updated` event is delivered when subscribed. [S5][S8]
2. Archive a disposable sandbox product and confirm it is no longer available for new checkouts; unarchive it only if the test needs to continue. [S5]
3. Create or update a disposable sandbox webhook endpoint, verify its event list and enabled state, then delete it when the test endpoint is no longer needed. The API exposes create, list, update, and delete operations with webhook scopes. [S19][S20][S22][S23]
4. Review token expiration dates and secret-store entries as part of each environment change. Replace credentials deliberately, verify the replacement with the read-only product check, and remove the old value from the deployment configuration. The OAT setup supports expiration and scoped permissions. [S4]

### Production go/no-go

1. Confirm the production organization is the reviewed organization, the owner/KYC and payout setup are complete, and the account is eligible for payouts. [S9][S10][S11]
2. Confirm the public website, support email, catalog product, production checkout path, and production webhook endpoint are ready. Polar's review guidance recommends a complete integration and live website for a clean review. [S9][S21]
3. Run a zero-value production canary only if needed, using a free product or 100% discount. Do not enter real card details as a test. [S9]
4. Inspect the production checkout result and webhook delivery history, then confirm the production endpoint is enabled and no sandbox ID, URL, token, or secret crossed the boundary. [S2][S7][S17]
5. Record the go-live owner, product IDs, endpoint IDs, token expirations, and rollback action: stop new checkouts using the applicable catalog control, disable the webhook endpoint, or replace the affected credential while investigating. Endpoint disablement and re-enablement are documented Polar controls; the ownership and rollback record are runbook decisions. [S4][S5][S7]

## Explicit Non-Goals

This note does not define application billing tables, subscription state machines, entitlement grants, access-control policy, order reconciliation logic, or webhook business handlers. It only defines the Polar-side prerequisites and checks needed before those application concerns are implemented.

## Sources

- [S1] Polar introduction and quick start: https://polar.sh/docs/introduction
- [S2] Polar sandbox environment: https://polar.sh/docs/integrate/sandbox
- [S3] Polar authentication: https://polar.sh/docs/integrate/authentication
- [S4] Polar Organization Access Tokens: https://polar.sh/docs/integrate/oat
- [S5] Polar products and catalog behavior: https://polar.sh/docs/features/products
- [S6] Polar webhook setup: https://polar.sh/docs/integrate/webhooks/endpoints
- [S7] Polar webhook delivery, validation, retries, and monitoring: https://polar.sh/docs/integrate/webhooks/delivery
- [S8] Polar webhook event catalog: https://polar.sh/docs/integrate/webhooks/events
- [S9] Polar account reviews: https://polar.sh/docs/merchant-of-record/account-reviews
- [S10] Polar payout accounts: https://polar.sh/docs/features/finance/accounts
- [S11] Polar supported countries: https://polar.sh/docs/merchant-of-record/supported-countries
- [S12] Polar local webhook development: https://polar.sh/docs/integrate/webhooks/locally
- [S13] Polar team management and roles: https://polar.sh/docs/features/team-management
- [S14] Polar product API create reference: https://polar.sh/docs/api-reference/current/products/create-product
- [S15] Polar Checkout Links: https://polar.sh/docs/features/checkout/links
- [S16] Polar Checkout API: https://polar.sh/docs/features/checkout/session
- [S17] Polar API overview and base URLs: https://polar.sh/docs/api-reference/current/introduction
- [S18] Polar product list API reference: https://polar.sh/docs/api-reference/current/products/list-products
- [S19] Polar webhook create API reference: https://polar.sh/docs/api-reference/current/webhooks/create-webhook-endpoint
- [S20] Polar webhook list API reference: https://polar.sh/docs/api-reference/current/webhooks/list-webhook-endpoints
- [S21] Polar organization API reference: https://polar.sh/docs/api-reference/current/organizations/get-organization
- [S22] Polar webhook update API reference: https://polar.sh/docs/api-reference/current/webhooks/update-webhook-endpoint
- [S23] Polar webhook delete API reference: https://polar.sh/docs/api-reference/current/webhooks/delete-webhook-endpoint

[S1]: https://polar.sh/docs/introduction
[S2]: https://polar.sh/docs/integrate/sandbox
[S3]: https://polar.sh/docs/integrate/authentication
[S4]: https://polar.sh/docs/integrate/oat
[S5]: https://polar.sh/docs/features/products
[S6]: https://polar.sh/docs/integrate/webhooks/endpoints
[S7]: https://polar.sh/docs/integrate/webhooks/delivery
[S8]: https://polar.sh/docs/integrate/webhooks/events
[S9]: https://polar.sh/docs/merchant-of-record/account-reviews
[S10]: https://polar.sh/docs/features/finance/accounts
[S11]: https://polar.sh/docs/merchant-of-record/supported-countries
[S12]: https://polar.sh/docs/integrate/webhooks/locally
[S13]: https://polar.sh/docs/features/team-management
[S14]: https://polar.sh/docs/api-reference/current/products/create-product
[S15]: https://polar.sh/docs/features/checkout/links
[S16]: https://polar.sh/docs/features/checkout/session
[S17]: https://polar.sh/docs/api-reference/current/introduction
[S18]: https://polar.sh/docs/api-reference/current/products/list-products
[S19]: https://polar.sh/docs/api-reference/current/webhooks/create-webhook-endpoint
[S20]: https://polar.sh/docs/api-reference/current/webhooks/list-webhook-endpoints
[S21]: https://polar.sh/docs/api-reference/current/organizations/get-organization
[S22]: https://polar.sh/docs/api-reference/current/webhooks/update-webhook-endpoint
[S23]: https://polar.sh/docs/api-reference/current/webhooks/delete-webhook-endpoint
