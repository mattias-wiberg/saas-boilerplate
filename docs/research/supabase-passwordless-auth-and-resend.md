# Research: Supabase Passwordless Auth and Resend Delivery

- Ticket: [GitHub issue #2](https://github.com/mattias-wiberg/saas-boilerplate/issues/2)
- Mode: AFK research
- Branch: `research/supabase-passwordless-auth-and-resend`
- Date: 2026-09-02
- Scope: Supabase email passwordless authentication, Next.js 16 App Router session handling, protected routes, redirects, Resend delivery, verification/recovery constraints, and test/live operations.
- Constraint: Research only. No application code or database changes are included.

## Recommended Decision Set

1. **Use Supabase email OTP as the starter's default passwordless experience.** It uses the same `signInWithOtp` entry point as a magic link, but a six-digit code avoids the single-use link prefetch problem documented by Supabase. Keep magic links as a later, explicit UX option rather than sending both modes in the baseline template. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless), [email templates](https://supabase.com/docs/guides/auth/auth-email-templates))
2. **Use `@supabase/ssr` with PKCE and request-scoped browser/server clients.** For a magic-link variant, use the documented token-hash confirmation endpoint; for OTP, verify the submitted email and code through the cookie-aware client. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs), [PKCE flow](https://supabase.com/docs/guides/auth/sessions/pkce-flow))
3. **Retain `proxy.ts` for token refresh, not as the only authorization boundary.** The Proxy should refresh the Supabase session and can perform cheap route redirects. Pages, Server Actions, Route Handlers, and data access must independently authorize requests; database access must also be protected with RLS. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs), [Next.js authentication](https://nextjs.org/docs/app/guides/authentication), [Next.js Proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy), [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security))
4. **Use Resend through Supabase Auth's custom SMTP integration.** Configure a verified auth sending domain and a least-privileged Resend sending key; no Resend SDK or API call is needed for the baseline Auth path. ([Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp), [Resend with Supabase SMTP](https://resend.com/docs/send-with-supabase-smtp), [Resend API keys](https://resend.com/docs/create-an-api-key))
5. **Keep the Resend API as an escalation path, not the default.** Use it through Supabase's Send Email Auth Hook only if custom rendering, queues, provider failover, or provider-specific API features justify an Edge Function and hook-secret lifecycle. ([Supabase Send Email Hook](https://supabase.com/docs/guides/auth/auth-hooks/send-email-hook), [Resend Send Email API](https://resend.com/docs/api-reference/emails/send-email))
6. **Do not add password recovery to the passwordless baseline.** Email OTP/magic-link sign-in is the recovery path. If passwords are added later, implement Supabase's separate recovery flow and protected change-password page; the recovery email shares the same expiration setting as OTPs and magic links. ([Supabase password-based Auth](https://supabase.com/docs/guides/auth/passwords), [passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless))
7. **Use local Mailpit for development, a separately controlled staging delivery setup, and verified custom SMTP in production.** The hosted default sender is intentionally restricted, rate-limited, and best-effort. ([Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp), [Supabase password-based Auth](https://supabase.com/docs/guides/auth/passwords))

## Findings

### Passwordless Options

Supabase documents two email passwordless methods: Magic Link and email OTP. Both are initiated with `signInWithOtp`; the default email is a Magic Link, while an OTP is produced by changing the Magic Link/OTP email template to include `{{ .Token }}`. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless), [email templates](https://supabase.com/docs/guides/auth/auth-email-templates))

| Option | How it works | Constraints and tradeoffs |
| --- | --- | --- |
| Magic Link | Send the user a one-click link. The link is one-time use and the user is signed in after it is verified. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless)) | A user can request one only every 60 seconds by default and it expires after one hour. Email security scanners or link tracking can consume or rewrite a one-time link before the user clicks it. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless), [email templates](https://supabase.com/docs/guides/auth/auth-email-templates), [production checklist](https://supabase.com/docs/guides/deployment/going-into-prod)) |
| Email OTP | Send a six-digit `{{ .Token }}` and verify it with the user's email, token, and type `email`. A successful verification creates a session. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless), [email templates](https://supabase.com/docs/guides/auth/auth-email-templates)) | It has the same default 60-second resend window and one-hour expiry. It adds a code-entry step, but avoids making the authentication action depend on a mail client or link scanner preserving a one-time URL. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless), [email templates](https://supabase.com/docs/guides/auth/auth-email-templates)) |

The template variables are material to the design: `{{ .ConfirmationURL }}` is the generated confirmation URL, `{{ .Token }}` is the six-digit OTP, and `{{ .TokenHash }}` is the hash used to construct a server-side confirmation link. ([email templates](https://supabase.com/docs/guides/auth/auth-email-templates))

Calling `signInWithOtp` auto-creates a user by default when the email is new. Set `shouldCreateUser` to `false` when a login form must not create accounts; leave creation enabled only when the product deliberately combines first sign-in and onboarding. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless))

Email authentication, including Magic Links and email OTP, is enabled by default. On hosted Supabase projects, email confirmation is enabled by default for email/password authentication, and Supabase says an unverified email identity cannot sign in by default. Keep confirmation enabled in live environments unless there is a documented product and security reason not to. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless), [Supabase users](https://supabase.com/docs/guides/auth/users), [Supabase password-based Auth](https://supabase.com/docs/guides/auth/passwords))

Supabase's June 2026 changelog says new Free-plan projects using the default SMTP service cannot customize Auth email templates; custom SMTP restores template customization. This makes the custom SMTP decision operationally necessary for an OTP template on those projects, not only a production-delivery improvement. ([Supabase changelog: Auth email template customization](https://supabase.com/changelog/46599-changes-to-custom-email-template-customisation-on-free-tier), [Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp))

### Next.js 16 Session Flow

The current Next.js documentation identifies version 16.3.4 and uses the `proxy.ts` file convention. Next.js 16 renamed the `middleware` convention to `proxy`; a root-level `proxy.ts` runs before routes and may redirect or modify request and response cookies. ([Next.js Proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy))

Supabase's current Next.js SSR guide requires two clients: a browser client for Client Components and a server client for Server Components, Server Actions, and Route Handlers. The guide says `@supabase/ssr` defaults to PKCE, stores session information in cookies, and requires a Proxy because Server Components cannot write cookies. The Proxy refreshes the Auth token, forwards the refreshed token to Server Components through request cookies, and writes the refreshed token back through response cookies. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs), [Supabase advanced SSR guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide))

Use the Supabase auth methods for their documented jobs:

- Use `getClaims()` to validate the JWT for routine page and data protection. The current guide says it validates the JWT signature using the project's published keys when asymmetric signing keys are in use. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs))
- Use `getUser()` when an up-to-date user record or an Auth-server network check is required. This is the appropriate stronger check for flows that must observe server-side user state rather than only a locally valid JWT. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs), [Supabase advanced SSR guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide))
- Use `getSession()` when the raw access token, refresh token, or expiry is needed. Do not use the user object returned by `getSession()` alone for authorization in server code, because the session can be loaded from storage without re-validating the user with Auth. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs))

Next.js 16's `cookies()` API is asynchronous. Server Components may read incoming cookies, but cookie writes and deletes must happen in a Server Function or Route Handler; cookies cannot be set after streaming starts. The Proxy API separately exposes incoming request cookies and outgoing response cookies. ([Next.js cookies](https://nextjs.org/docs/app/api-reference/functions/cookies), [Next.js Proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy))

Do not create a second hand-rolled session cookie around the Supabase session. `@supabase/ssr` supplies the request-aware cookie adapter, including cache headers when it refreshes a token. Apply those headers to the Proxy response; otherwise a CDN or ISR response containing `Set-Cookie` can leak one user's refreshed session to another user. Supabase also recommends initializing the client inside the request handler rather than in shared module state. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs), [Supabase advanced SSR guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide))

The generic Next.js authentication guide recommends secure cookie attributes for a custom session, including `HttpOnly`, `Secure`, `SameSite`, and an expiry. Supabase's SSR guide is the governing source for this integration: its advanced guide says `HttpOnly` is not required for the `@supabase/ssr` browser/server model because browser-side code needs access to the refresh token. Use the helper's cookie strategy rather than adding a conflicting custom cookie policy. ([Next.js authentication](https://nextjs.org/docs/app/guides/authentication), [Supabase advanced SSR guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide))

Supabase sessions contain a short-lived access token and a refresh token. The current sessions guide says access tokens are usually valid for five minutes to one hour, refresh tokens do not expire but are single-use with a reuse interval, and the default session remains active until sign-out or another terminating event. The guide recommends one hour as the normal JWT expiry and warns against going below five minutes for most applications. ([Supabase user sessions](https://supabase.com/docs/guides/auth/sessions))

### Magic-Link and OTP Callback Shape

For SSR, do not rely on an implicit-flow URL fragment: browsers do not send fragments to the server. Supabase's PKCE flow sends an Auth Code in a query parameter, and that code is valid for five minutes and can be exchanged only once. The code verifier is stored locally, so a normal PKCE exchange must use the same browser and device that initiated the flow; overlapping flows can overwrite the stored verifier. ([Supabase implicit flow](https://supabase.com/docs/guides/auth/sessions/implicit-flow), [Supabase PKCE flow](https://supabase.com/docs/guides/auth/sessions/pkce-flow))

For a Next.js SSR Magic Link, the current Supabase passwordless guide instead documents a custom email link containing `{{ .TokenHash }}` and `type=email`, directed to an `/auth/confirm` Route Handler. That handler verifies the hash with `verifyOtp`, which establishes the cookie-backed session before redirecting to the destination. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless), [Supabase password-based Auth PKCE example](https://supabase.com/docs/guides/auth/passwords))

For email OTP, render a code-entry state after the send call, then verify the email and six-digit token with `type=email`. A successful send intentionally returns no session yet; the session arrives only after verification. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless))

### Protected Routes and Redirects

Next.js describes Proxy checks as optimistic checks useful for pre-filtering routes and redirects, but says the majority of security checks belong close to the data source. It specifically warns that layouts do not stop nested route segments or Server Actions, and that Server Actions and Route Handlers must perform their own authorization checks. ([Next.js authentication](https://nextjs.org/docs/app/guides/authentication), [Next.js Proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy))

The protected-route boundary should therefore be layered:

1. **Proxy:** refresh the Supabase session on requests that need it, and optionally redirect clearly unauthenticated users from known protected paths. Keep the matcher broad enough to cover the routes and Server Functions that need session refresh, while excluding static assets. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs), [Next.js Proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy))
2. **Server access layer:** call `getClaims()` or the stronger `getUser()` check at the page/data/action/handler boundary. Do not treat a successful render of a parent layout as authorization. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs), [Next.js authentication](https://nextjs.org/docs/app/guides/authentication))
3. **Database:** enforce tenant ownership with RLS for any user data reached through Supabase's Data API. A route redirect alone is not a database authorization policy. ([Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security))

Supabase requires the Site URL and additional redirect URLs to be configured before a passwordless `redirectTo` can be used. The Site URL is the fallback destination; Supabase recommends exact production paths and allows wildcards for local or preview environments. ([Supabase redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls), [passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless))

The callback should accept only a small, known set of internal `next` destinations. This is a recommendation to narrow the callback's navigation policy: Supabase's allowlist controls Auth redirect destinations, while Next.js `redirect()` also accepts absolute URLs. ([Supabase redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls), [Next.js redirect](https://nextjs.org/docs/app/api-reference/functions/redirect))

Next.js `redirect()` can be used in Server Components, Route Handlers, and Server Functions. It throws to terminate rendering, returns a 307-style temporary redirect in normal contexts, and uses a 303 for progressive-enhancement Server Action submissions. Keep it outside `try` blocks when catching other errors. ([Next.js redirect](https://nextjs.org/docs/app/api-reference/functions/redirect))

### Database and RLS Boundary

Supabase says an exposed-schema table without RLS can be accessed by any role that has a grant, and that adding policies does not remove existing grants. Enable RLS and set grants for every exposed table; treat grants and policies as separate controls. ([Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security))

For this repository's baseline one-tenant-per-user model, the future data tables should use an owner key such as `user_id` and policies that compare it to `(select auth.uid())` for the `authenticated` role. Inserts need a `WITH CHECK` ownership condition; updates need both an existing-row `USING` condition and a resulting-row `WITH CHECK` condition so a user cannot reassign ownership. A corresponding SELECT policy is required for an UPDATE to work as expected. ([Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security), [RLS basics](https://supabase.com/docs/guides/database/postgres/row-level-security#rls-reference))

Index columns used by RLS predicates, and test allowed and denied operations with Supabase's database test workflow. Supabase recommends wrapping stable helper calls such as `auth.uid()` in a SELECT subquery for policy performance. ([Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security), [Supabase Postgres RLS performance guidance](https://supabase.com/docs/guides/database/postgres/row-level-security#call-functions-with-select))

Do not use editable `user_metadata` as an authorization source. Supabase identifies `raw_user_meta_data` as user-editable and recommends application metadata for authorization claims, with the additional caveat that JWT claims are not refreshed until the token is refreshed. ([Supabase users](https://supabase.com/docs/guides/auth/users), [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security))

No tables, policies, migrations, or tests are part of this research ticket.

### Verification and Recovery

The Email OTP Expiration setting is shared: Supabase says it controls email OTPs, Magic Links, email confirmation links, password recovery links, email-change links, and invitations. Its default is one hour; durations over 86,400 seconds are strongly discouraged, and the production checklist recommends 3,600 seconds or lower. Shortening it therefore changes more than sign-in behavior. ([passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless), [Supabase production checklist](https://supabase.com/docs/guides/deployment/going-into-prod))

Magic Links and Auth email links are single-use. Mail security scanners may issue a GET before the user sees the message, consuming the link; Supabase recommends using an OTP or putting the real confirmation URL behind a user-clicked intermediate page. Supabase also recommends disabling external email link tracking because tracking can overwrite or deform Auth links. ([email templates](https://supabase.com/docs/guides/auth/auth-email-templates), [Supabase production checklist](https://supabase.com/docs/guides/deployment/going-into-prod), [Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp))

Password recovery is a separate flow. `resetPasswordForEmail()` does not reveal whether an account exists, and Supabase's PKCE example sends a recovery token hash to `/auth/confirm` with `type=recovery`; after verification, the user reaches an authenticated change-password page and calls `updateUser`. For a passwordless-only product, omit this flow and let the user request a fresh email OTP instead. ([Supabase password-based Auth](https://supabase.com/docs/guides/auth/passwords))

If password recovery is added later, cover these states explicitly: unknown email with the same user-facing response, expired or reused recovery token, invalid redirect, successful session establishment, and an authenticated password update. The first behavior is an anti-enumeration property documented by Supabase; the remaining cases follow the documented token and protected-page flow. ([Supabase password-based Auth](https://supabase.com/docs/guides/auth/passwords), [Supabase email templates](https://supabase.com/docs/guides/auth/auth-email-templates))

### Delivery: Supabase Auth, Resend SMTP, and Resend API

The baseline delivery path is:

`Supabase Auth -> custom SMTP -> smtp.resend.com -> recipient`

Supabase Auth supports any SMTP provider and uses the configured host, port, username, password, and sender address for Auth messages. Supabase explicitly lists Resend as a compatible provider. ([Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp))

Resend's Supabase integration documents these credentials: host `smtp.resend.com`, port `465`, username `resend`, and the Resend API key as the password. A verified domain and API key are prerequisites. After saving the settings in Supabase, Auth mail is sent through Resend. ([Resend with Supabase SMTP](https://resend.com/docs/send-with-supabase-smtp), [Resend SMTP](https://resend.com/docs/send-with-smtp), [Resend verified domains](https://resend.com/docs/dashboard/domains/introduction))

The Resend API is a different delivery interface. It uses HTTPS at `https://api.resend.com`, authenticates with a Bearer API key, and accepts an email payload at the Send Email API. SMTP and API messages still appear in Resend's email table, and Resend says SMTP uses the same rate limit as its API. ([Resend API introduction](https://resend.com/docs/api-reference/introduction), [Resend Send Email API](https://resend.com/docs/api-reference/emails/send-email), [Resend SMTP](https://resend.com/docs/send-with-smtp))

Supabase's Send Email Auth Hook replaces the normal SMTP send path. It exposes the Auth event, including the OTP token, token hash, redirect URL, and action type, to a hook; the official guide shows using an Edge Function and Resend API for custom sending. This is the correct API relationship if the product later needs custom templates, queuing, failover, or provider-specific controls. ([Supabase Send Email Hook](https://supabase.com/docs/guides/auth/auth-hooks/send-email-hook))

Use a separate verified auth subdomain and sender from marketing mail, disable open/click tracking for authentication messages, and configure SPF, DKIM, and DMARC with the sending provider. Supabase recommends separating Auth and marketing reputation; Resend recommends verified subdomains for reputation segmentation and requires DNS records for domain verification. ([Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp), [Resend add and verify a domain](https://resend.com/docs/add-a-domain), [Resend verified domains](https://resend.com/docs/dashboard/domains/introduction))

Create a Resend key with sending access and restrict it to the auth domain where possible. Resend describes API keys as secret, says a sending-access key can be restricted to a specific domain, and says the key is only shown once. Store it only in the Supabase SMTP configuration or hook/server secret store, never in browser code. ([Resend API keys](https://resend.com/docs/create-an-api-key), [Supabase API keys](https://supabase.com/docs/guides/getting-started/api-keys))

### Test and Live Concerns

| Area | What to verify | Official constraint |
| --- | --- | --- |
| Local email | Run the Auth flow against a local Supabase stack and inspect Mailpit rather than depending on external delivery. | The Supabase CLI captures local emails with Mailpit; `supabase status` reports the Mailpit URL. ([Supabase password-based Auth](https://supabase.com/docs/guides/auth/passwords)) |
| Hosted default email | Do not use the default sender as a production delivery service. | It sends only to pre-authorized project-team addresses, is currently limited to two messages per hour, and has no delivery or uptime SLA. ([Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp)) |
| Supabase Auth limits | Test resend cooldowns and abuse responses, not only successful sign-in. | The current rate-limit guide documents a 60-second per-user window for OTP/magic-link requests and a default project-wide limit of 30 OTPs per hour; exceeded limits return 429. ([Supabase rate limits](https://supabase.com/docs/guides/auth/rate-limits)) |
| Custom SMTP startup | Confirm the project-level Auth limit is appropriate before launch or a campaign. | Supabase imposes a low 30-message-per-hour limit after custom SMTP setup and directs projects to adjust it in Auth rate-limit settings. ([Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp)) |
| Resend capacity | Monitor provider responses and the Resend email table. | Resend's default API maximum is 10 requests per second per team, and the SMTP path uses the same API rate limit. ([Resend API introduction](https://resend.com/docs/api-reference/introduction), [Resend SMTP](https://resend.com/docs/send-with-smtp)) |
| Domain and sender | Verify the auth sending domain, DNS records, From address, and TLS connection before a live user test. | Resend requires a verified domain and documents SMTP TLS options; it recommends a subdomain and DNS verification. ([Resend SMTP](https://resend.com/docs/send-with-smtp), [Resend add and verify a domain](https://resend.com/docs/add-a-domain)) |
| Redirects | Test localhost, staging, production, callback failure, and a missing or disallowed `next`. | Supabase accepts only configured redirect URLs and recommends exact production paths; failed Auth redirects can return error details in URL fragments. ([Supabase redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls)) |
| Cookie refresh | Verify sign-in, a later request after access-token expiry, sign-out, refresh failure, and cache behavior. | Supabase requires Proxy request/response cookie handling for refresh and warns that cached `Set-Cookie` responses can leak sessions. ([Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs), [Supabase advanced SSR guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide)) |
| Link robustness | Test link prefetch, duplicate clicks, expired links, link tracking, and the OTP alternative. | Auth links are one-time use; Supabase documents scanners and tracking as failure sources and recommends OTP or an intermediate click. ([Supabase email templates](https://supabase.com/docs/guides/auth/auth-email-templates), [Supabase production checklist](https://supabase.com/docs/guides/deployment/going-into-prod)) |
| PKCE flows | Test same-browser completion, a second tab, expired code, and a new request after failure. | PKCE codes are single-use and valid for five minutes; the verifier is local and overlapping flows can replace it. ([Supabase PKCE flow](https://supabase.com/docs/guides/auth/sessions/pkce-flow)) |
| Data isolation | Test anonymous, owner, and different-user reads/writes against every user-owned table. | Supabase requires RLS and grants to be configured separately and recommends database tests for allowed and denied operations. ([Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security)) |

Before going live, also verify that the production Site URL and redirect allowlist are exact, auth mail uses a verified domain, link tracking is disabled, the Resend key is restricted and secret, and authenticated responses cannot be cached across users. These are configuration gates, not application-code work for this ticket. ([Supabase redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls), [Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp), [Resend API keys](https://resend.com/docs/create-an-api-key), [Supabase advanced SSR guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide))

## Sources

All sources consulted are first-party documentation from Supabase, Next.js, or Resend:

- [Supabase passwordless email logins](https://supabase.com/docs/guides/auth/auth-email-passwordless)
- [Supabase email templates](https://supabase.com/docs/guides/auth/auth-email-templates)
- [Supabase Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs)
- [Supabase advanced SSR guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide)
- [Supabase user sessions](https://supabase.com/docs/guides/auth/sessions)
- [Supabase PKCE flow](https://supabase.com/docs/guides/auth/sessions/pkce-flow)
- [Supabase implicit flow](https://supabase.com/docs/guides/auth/sessions/implicit-flow)
- [Supabase redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls)
- [Supabase rate limits](https://supabase.com/docs/guides/auth/rate-limits)
- [Supabase password-based Auth](https://supabase.com/docs/guides/auth/passwords)
- [Supabase custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp)
- [Supabase Send Email Hook](https://supabase.com/docs/guides/auth/auth-hooks/send-email-hook)
- [Supabase users](https://supabase.com/docs/guides/auth/users)
- [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase production checklist](https://supabase.com/docs/guides/deployment/going-into-prod)
- [Supabase changelog: Auth email template customization](https://supabase.com/changelog/46599-changes-to-custom-email-template-customisation-on-free-tier)
- [Next.js authentication](https://nextjs.org/docs/app/guides/authentication)
- [Next.js Proxy](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)
- [Next.js cookies](https://nextjs.org/docs/app/api-reference/functions/cookies)
- [Next.js redirect](https://nextjs.org/docs/app/api-reference/functions/redirect)
- [Resend SMTP](https://resend.com/docs/send-with-smtp)
- [Resend with Supabase SMTP](https://resend.com/docs/send-with-supabase-smtp)
- [Resend verified domains](https://resend.com/docs/dashboard/domains/introduction)
- [Resend add and verify a domain](https://resend.com/docs/add-a-domain)
- [Resend API keys](https://resend.com/docs/create-an-api-key)
- [Resend API introduction](https://resend.com/docs/api-reference/introduction)
- [Resend Send Email API](https://resend.com/docs/api-reference/emails/send-email)
