# Supabase Local-to-Production Workflow

**Research date:** 2026-09-02
**Scope:** local development, database migrations, seed data, RLS verification, secrets, Auth configuration, and promotion to one hosted production project.

## Decision Summary

1. **Develop against the local Supabase stack.** Initialize the repository with the Supabase CLI, run the stack in Docker, and never expose the local stack to external traffic. The local stack is for development and testing, not production. ([CLI getting started](https://supabase.com/docs/guides/local-development/cli/getting-started), [local workflow](https://supabase.com/docs/guides/local-development/cli-workflows))
2. **Make versioned SQL migrations the deployment contract.** For this starter, prefer explicit imperative migrations so RLS, grants, and data-shape changes are visible in review. Supabase also supports declarative schemas, and currently recommends them for new projects; choose one model and do not maintain both. Declarative files become the source of truth, while schema diff does not capture DML and has RLS/view caveats. ([declarative schemas](https://supabase.com/docs/guides/local-development/declarative-database-schemas), [database migrations](https://supabase.com/docs/guides/deployment/database-migrations))
3. **Keep seed data local-only.** Use `supabase/seed.sql` for representative development/test data. It runs after migrations on `supabase start` and `supabase db reset`; never use `--include-seed` against production. ([seeding](https://supabase.com/docs/guides/local-development/seeding-your-database), [CLI `db push`](https://supabase.com/docs/reference/cli/supabase-db-push))
4. **Treat RLS and grants as one security change.** Enable RLS on every exposed application table, grant only the operations each role needs, name roles explicitly in policies, and add pgTAP tests for both allowed and denied operations. ([RLS](https://supabase.com/docs/guides/auth/row-level-security), [database testing](https://supabase.com/docs/guides/database/testing))
5. **Use separate local and hosted credentials.** Browser-facing code receives a publishable key; server-only code may receive a secret key. Local keys are unrelated to hosted keys. The repository currently uses the legacy `NEXT_PUBLIC_SUPABASE_ANON_KEY`; future implementation work should plan the migration to `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, since Supabase says the legacy `anon` and `service_role` keys are being deprecated by the end of 2026. ([API keys](https://supabase.com/docs/guides/getting-started/api-keys), [Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs))
6. **Promote reviewed migrations from `main` to the single production project.** Run local database tests in CI, then use Supabase GitHub integration or a guarded GitHub Actions job to apply migrations to production. Keep the Next.js deployment separate but on the same release boundary. Supabase's production integration applies migrations, functions, and declared storage buckets, but ignores Auth, API, and seed configuration by default. ([managing environments](https://supabase.com/docs/guides/deployment/managing-environments), [GitHub integration](https://supabase.com/docs/guides/deployment/branching/github-integration), [production checklist](https://supabase.com/docs/guides/deployment/going-into-prod))

The one-project constraint means there is no hosted staging database in this workflow. Local Docker plus CI pgTAP is the minimum gate; high-risk changes should add a temporary preview or staging environment later rather than weakening the production safeguards. ([managing environments](https://supabase.com/docs/guides/deployment/managing-environments), [branching](https://supabase.com/docs/guides/deployment/branching))

## Repository Baseline

This repository is a minimal Next.js `16.3.4` starter using pnpm. It currently has:

- `@supabase/ssr` and `@supabase/supabase-js` dependencies in `package.json`.
- Browser, server, and middleware clients under `utils/supabase/`.
- `proxy.ts` calling the middleware client on application requests.
- `NEXT_PUBLIC_SUPABASE_URL` and legacy `NEXT_PUBLIC_SUPABASE_ANON_KEY` references in all three clients.
- No `supabase/` directory, migrations, seeds, database tests, or Supabase CLI dependency.
- `.env*` ignored by `.gitignore`.

This note does not change application code or bootstrap the Supabase directory. Those are follow-up implementation work.

The current middleware calls `supabase.auth.getUser()`. The current SSR guide recommends `getClaims()` for protecting pages and user data and warns not to use `getSession()` for server authorization. Reconcile that guidance in a separate application change; it is outside this research ticket. ([Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs))

## Initial Setup

The CLI supports global installation or a project dependency. A project dependency should be pinned and invoked through the package runner; the npm/pnpm path requires Node.js 20 or later. The local stack requires a Docker-compatible runtime. ([CLI getting started](https://supabase.com/docs/guides/local-development/cli/getting-started))

For this pnpm repository, the future implementation path is:

```bash
pnpm add -D supabase
pnpm supabase --help
pnpm supabase init
```

`supabase init` creates `supabase/config.toml`. Commit `config.toml`, migrations, tests, and seed files. Do not commit values placed in `.env` files. The config file is safe to commit unless secrets are hardcoded into it. ([local workflow](https://supabase.com/docs/guides/local-development/cli-workflows), [config and secrets](https://supabase.com/docs/guides/local-development/managing-config))

Start and inspect the local stack with:

```bash
pnpm supabase start
pnpm supabase status
```

The default local API is `http://127.0.0.1:54321`, Studio is `http://127.0.0.1:54323`, Mailpit is `http://127.0.0.1:54324`, and the local database is on port `54322`. `supabase start` prints local publishable and secret keys; `supabase status` can print the connection details again. Local keys do not grant access to the hosted project. ([CLI getting started](https://supabase.com/docs/guides/local-development/cli/getting-started), [API keys](https://supabase.com/docs/guides/getting-started/api-keys))

The local stack is not hardened: it has no TLS, no rate limiting, and default credentials. Do not expose it to external traffic. ([local workflow](https://supabase.com/docs/guides/local-development/cli-workflows))

## Existing Project Bootstrap

If the single hosted project already contains schema changes, establish a baseline before writing new migrations:

```bash
pnpm supabase login
pnpm supabase link --project-ref <project-ref>
pnpm supabase db pull
```

`db pull` creates a migration under `supabase/migrations/` and can record that baseline in the remote migration history. Review the generated SQL before committing it. Auth and Storage schemas are excluded by default and should only be pulled separately when custom changes require it. ([local workflow](https://supabase.com/docs/guides/local-development/cli-workflows), [CLI `db pull`](https://supabase.com/docs/reference/cli/supabase-db-pull))

After the baseline is committed, all future remote schema changes should be represented by migration files. Do not edit the hosted database through the Dashboard or SQL editor once migration history is established. If drift already exists, inspect it with `supabase migration list`, capture genuine schema changes with `supabase db pull`, and use `supabase migration repair` only when the database state is already correct. ([database migrations](https://supabase.com/docs/guides/deployment/database-migrations), [CLI migration reference](https://supabase.com/docs/reference/cli))

If the hosted project is empty, skip `db pull` and create the first migration locally.

## Daily Migration Workflow

For imperative SQL migrations:

```bash
pnpm supabase migration new <descriptive-name>
# edit supabase/migrations/<timestamp>_<descriptive-name>.sql
pnpm supabase db reset
pnpm supabase test db
```

`migration new` creates a timestamped file. `db reset` rebuilds the local database from all migrations and then runs the seed file. `migration up` can apply only pending local migrations, but `db reset` is the stronger reproducibility check because it exercises the complete chain. ([database migrations](https://supabase.com/docs/guides/deployment/database-migrations), [CLI migration reference](https://supabase.com/docs/reference/cli))

If the team later chooses declarative schemas, edit only `supabase/schemas/`, generate a migration with `supabase db diff -f <name>`, review it, run `supabase db reset`, and commit the schema and generated migration together. Do not make direct Studio changes expecting declarative `db diff` to discover them. ([declarative schemas](https://supabase.com/docs/guides/local-development/declarative-database-schemas))

Before production deployment:

```bash
pnpm supabase db push --dry-run
pnpm supabase migration list
pnpm supabase db push
```

`db push` applies local migrations not present in the remote `supabase_migrations.schema_migrations` history. Coordinate so only one release process pushes at a time. Never use `supabase db reset --linked` against production; linked reset is destructive. ([CLI `db push`](https://supabase.com/docs/reference/cli/supabase-db-push), [CLI `db reset`](https://supabase.com/docs/reference/cli/supabase-db-reset), [database migrations](https://supabase.com/docs/guides/deployment/database-migrations))

Migrations should move production forward. If a deployed change needs correction, create a new migration rather than resetting or editing an already-applied migration. ([declarative schemas](https://supabase.com/docs/guides/local-development/declarative-database-schemas))

## Seed Data

Use `supabase/seed.sql` for safe, representative local data. Seed files run after all migrations on the first `supabase start` and every `supabase db reset`. Prefer insert-only seed content and keep production users, personal data, secrets, and credentials out of it. ([seeding](https://supabase.com/docs/guides/local-development/seeding-your-database), [local workflow](https://supabase.com/docs/guides/local-development/cli-workflows))

For this one-production-project workflow:

- Seed local and CI databases by default.
- Do not include production data in the committed seed file.
- Do not run `supabase db push --include-seed` against production.
- Do not use `supabase db reset --linked` against production.

The CLI supports `--include-seed` for remote pushes, but the workflow documentation explicitly limits that practice to development or staging environments. ([CLI `db push`](https://supabase.com/docs/reference/cli/supabase-db-push), [local workflow](https://supabase.com/docs/guides/local-development/cli-workflows))

## RLS and Database Verification

RLS is not a substitute for grants. Grants decide whether a role may perform an operation; policies decide which rows the operation may affect. Enable RLS on every table in an exposed schema, revoke client grants that are not needed, and grant back only the intended operations. ([RLS](https://supabase.com/docs/guides/auth/row-level-security))

For the baseline one-tenant-per-user domain in `CONTEXT.md`, a typical policy shape is:

```sql
alter table public.<table> enable row level security;

revoke all on table public.<table> from anon, authenticated;
grant select, insert, update, delete on table public.<table> to authenticated;

create policy "Users can read their own rows"
on public.<table>
for select
to authenticated
using ((select auth.uid()) = user_id);

create policy "Users can insert their own rows"
on public.<table>
for insert
to authenticated
with check ((select auth.uid()) = user_id);

create policy "Users can update their own rows"
on public.<table>
for update
to authenticated
using ((select auth.uid()) = user_id)
with check ((select auth.uid()) = user_id);

create policy "Users can delete their own rows"
on public.<table>
for delete
to authenticated
using ((select auth.uid()) = user_id);
```

The exact grants and policy expressions must follow each table's intended access. Name the target role with `to`; separate policies by operation; use `with check` to constrain inserted and updated rows; and remember that an update also needs a corresponding select policy. ([RLS](https://supabase.com/docs/guides/auth/row-level-security))

Create one pgTAP test file per protected table under `supabase/tests/database/` and run:

```bash
pnpm supabase test new <table>_rls.test
pnpm supabase test db
```

Tests should run inside a transaction and roll back. Cover anonymous and authenticated roles, allowed and denied select/insert/update/delete operations, ownership boundaries, and the integrity of rows targeted by denied writes. Use `set local role authenticated` and `set local request.jwt.claim.sub = '<user-id>'` to model users. A zero-row update is not proof of a denied write; assert the target row remains unchanged. ([database testing](https://supabase.com/docs/guides/database/testing), [RLS](https://supabase.com/docs/guides/auth/row-level-security), [CLI `test db`](https://supabase.com/docs/reference/cli/supabase-test-db))

Index columns used by policies, especially `user_id`, and wrap row-independent helper calls such as `auth.uid()` in `select` so Postgres can cache the result per statement. Measure RLS changes only in a non-production environment; disabling RLS for comparison exposes data. ([RLS performance](https://supabase.com/docs/guides/database/postgres/row-level-security-performance))

CI should start the local stack and run `supabase test db` on pull requests. The official testing guide demonstrates `supabase/setup-cli@v1`, `supabase start`, and `supabase test db`. ([database testing](https://supabase.com/docs/guides/database/testing))

## Environment and Secrets

Next.js loads `.env*` files into `process.env`, and the default Next.js template ignores them in Git. A variable prefixed with `NEXT_PUBLIC_` is inlined into browser JavaScript during `next build`; it must therefore be set to the correct environment before the production build. Non-public variables remain server-only. ([Next.js environment variables](https://nextjs.org/docs/app/guides/environment-variables))

Use this boundary:

| Consumer | Local value | Production value |
| --- | --- | --- |
| Browser and SSR Supabase clients | Local API URL plus local publishable key | Hosted project URL plus hosted publishable key |
| Server-only administrative code | No secret unless needed | Server-only secret key, never `NEXT_PUBLIC_*` |
| Supabase CLI in CI | CI environment secrets | CI environment secrets for the same production project |
| Supabase Edge Functions, if added later | `supabase/functions/.env` or `--env-file` | `supabase secrets set` or Dashboard secret storage |

Supabase defines publishable keys as safe for shipped code only when RLS controls access, while secret keys provide elevated access and bypass RLS when used without a user access token; they must remain in controlled server components. Local keys are separate from hosted keys. ([API keys](https://supabase.com/docs/guides/getting-started/api-keys))

Suggested variable names for future implementation are:

```dotenv
# Browser-safe and therefore exposed in the Next.js bundle.
NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=local-publishable-key

# Server-only; include only if the application needs privileged operations.
SUPABASE_URL=http://127.0.0.1:54321
SUPABASE_SECRET_KEY=local-secret-key
```

Use the hosted equivalents in the production app environment. Do not put a secret key in source control, browser code, a Docker image intended for client distribution, or a public document. ([API keys](https://supabase.com/docs/guides/getting-started/api-keys), [Next.js environment variables](https://nextjs.org/docs/app/guides/environment-variables))

For CLI deployment, store `SUPABASE_ACCESS_TOKEN`, `SUPABASE_DB_PASSWORD`, and `SUPABASE_PROJECT_ID` as encrypted CI secrets. Do not print them or place them in repository files. ([managing environments](https://supabase.com/docs/guides/deployment/managing-environments))

For local Supabase configuration, use `env(NAME)` in `supabase/config.toml` when an OAuth provider or other setting needs a secret. For Edge Functions, local values can come from `supabase/functions/.env` or `--env-file`, while hosted values can be set with `supabase secrets set`. ([config and secrets](https://supabase.com/docs/guides/local-development/managing-config), [Edge Function environment variables](https://supabase.com/docs/guides/functions/secrets))

## Auth URL and Configuration Boundaries

There are three distinct configuration surfaces:

1. **Local Auth:** `supabase/config.toml` controls local `auth.site_url` and `auth.additional_redirect_urls`. The local defaults target `http://localhost:3000`; restart the local stack after changing config. ([CLI config](https://supabase.com/docs/guides/local-development/cli/config))
2. **Hosted Auth:** the hosted project's Dashboard URL Configuration controls the production Site URL and allowed redirect URLs. `redirectTo` or `emailRedirectTo` must match the allow list. Set the hosted Site URL to the real production app URL and use exact production redirect paths; add local URLs only when local development points at the hosted Auth project. ([redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls), [password Auth](https://supabase.com/docs/guides/auth/passwords))
3. **Next.js app environment:** local and production app builds receive different public Supabase URL/key values. Because public values are build-time inlined, a single already-built client bundle cannot be promoted between environments and still acquire a different `NEXT_PUBLIC_*` value automatically. ([Next.js environment variables](https://nextjs.org/docs/app/guides/environment-variables))

The recommended default is to point local Next.js development at the local Supabase API and local Auth, and production Next.js at the one hosted project. This avoids accidentally writing development data into production. If local development must use hosted Auth, explicitly allow the local callback URLs in the hosted project and understand that local accounts/data then live in production.

Hosted email templates, SMTP, Auth providers, and URL settings are operational configuration. Local email templates and Auth settings can be represented in local config/files, but the Dashboard template builder does not apply to the local CLI stack. Supabase's GitHub production integration ignores Auth and API configuration by default, so do not treat a committed local `config.toml` as a complete hosted-project configuration system. ([email templates](https://supabase.com/docs/guides/auth/auth-email-templates), [CLI config](https://supabase.com/docs/guides/local-development/cli/config), [GitHub integration](https://supabase.com/docs/guides/deployment/branching/github-integration))

The SSR client uses the PKCE flow and cookies by default. The Next.js Proxy refreshes sessions; authenticated responses must not be cached in a way that can share a `Set-Cookie` response between users. ([SSR advanced guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide), [Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs))

## Promotion and Deployment

The proposed release path is:

```text
feature branch
  -> pull request
  -> local Supabase start/reset + pgTAP tests in CI
  -> merge to main
  -> guarded migration deployment to the one hosted production project
  -> Next.js production build and deployment with production env values
```

For a Supabase GitHub integration, enable production deployment and require its check before merging. The integration applies new migrations and selected Supabase assets when the production branch changes, but does not automatically apply Auth/API/seed configuration. ([GitHub integration](https://supabase.com/docs/guides/deployment/branching/github-integration), [production checklist](https://supabase.com/docs/guides/deployment/going-into-prod))

If explicit GitHub Actions is preferred, the production job needs the three encrypted Supabase variables and follows this shape:

```yaml
env:
  SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
  SUPABASE_DB_PASSWORD: ${{ secrets.PRODUCTION_DB_PASSWORD }}
  SUPABASE_PROJECT_ID: ${{ secrets.PRODUCTION_PROJECT_ID }}

steps:
  - uses: actions/checkout@v4
  - uses: supabase/setup-cli@v1
    with:
      version: "x.y.z"
  - run: supabase link --project-ref $SUPABASE_PROJECT_ID
  - run: supabase db push
```

Pin the CLI version in the repository or action rather than silently changing it on every release. If a manual production push is temporarily necessary, run `db push --dry-run` first and record the release; CI is preferred to a developer laptop. ([managing environments](https://supabase.com/docs/guides/deployment/managing-environments), [CLI getting started](https://supabase.com/docs/guides/local-development/cli/getting-started))

Next.js can run as a Node.js server or Docker container. The current starter already has `build` and `start` scripts, so the eventual host must set production environment values before `pnpm build` and run the resulting server with `pnpm start`. ([Next.js deployment](https://nextjs.org/docs/app/getting-started/deploying))

Apply a migration before deploying application code that requires its new schema. Keep schema and application changes in one reviewed release when possible, and use a forward migration to correct production rather than editing applied history. This ordering is a release policy for the one-project setup, not an automatic guarantee supplied by Supabase.

## Operational Checklist

- Pin and install the Supabase CLI as a project dev dependency when implementation begins.
- Confirm Node.js 20+ and a Docker-compatible runtime are available for local and CI work.
- Run `supabase init` at the repository root and commit the non-secret `supabase/` files.
- If a hosted schema already exists, link and pull a reviewed baseline before creating new migrations.
- Keep one migration model; for this note, use explicit imperative SQL migrations.
- Run `supabase db reset` after every migration change.
- Add per-table pgTAP tests for RLS allow and deny cases and run `supabase test db` in CI.
- Keep seed data representative and local-only.
- Use local URL/key values in local Next.js env files and hosted URL/key values in the production build environment.
- Keep secret keys, database passwords, and CLI access tokens in host/CI secret storage.
- Configure hosted Auth URLs, providers, SMTP, and templates separately from local `config.toml`.
- Deploy migrations only from the protected production branch; never reset or seed the production database.
- Review Supabase Security Advisor and enable production protections such as SSL enforcement, network restrictions, email confirmations, and custom SMTP as applicable. ([production checklist](https://supabase.com/docs/guides/deployment/going-into-prod))

## Primary Sources

- [Supabase local development workflow](https://supabase.com/docs/guides/local-development/cli-workflows)
- [Install and run the Supabase CLI](https://supabase.com/docs/guides/local-development/cli/getting-started)
- [Supabase CLI reference](https://supabase.com/docs/reference/cli)
- [Database migrations](https://supabase.com/docs/guides/deployment/database-migrations)
- [Declarative database schemas](https://supabase.com/docs/guides/local-development/declarative-database-schemas)
- [Seeding your database](https://supabase.com/docs/guides/local-development/seeding-your-database)
- [Testing your database](https://supabase.com/docs/guides/database/testing)
- [Row Level Security](https://supabase.com/docs/guides/auth/row-level-security)
- [RLS performance](https://supabase.com/docs/guides/database/postgres/row-level-security-performance)
- [Managing config and secrets](https://supabase.com/docs/guides/local-development/managing-config)
- [Supabase CLI config](https://supabase.com/docs/guides/local-development/cli/config)
- [Supabase API keys](https://supabase.com/docs/guides/getting-started/api-keys)
- [Auth redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls)
- [Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs)
- [SSR advanced guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide)
- [Managing environments](https://supabase.com/docs/guides/deployment/managing-environments)
- [Supabase GitHub integration](https://supabase.com/docs/guides/deployment/branching/github-integration)
- [Supabase production checklist](https://supabase.com/docs/guides/deployment/going-into-prod)
- [Supabase Edge Function environment variables](https://supabase.com/docs/guides/functions/secrets)
- [Next.js environment variables](https://nextjs.org/docs/app/guides/environment-variables)
- [Next.js deployment](https://nextjs.org/docs/app/getting-started/deploying)
