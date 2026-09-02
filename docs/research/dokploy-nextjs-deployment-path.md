# Research: Dokploy Next.js Deployment Path

**Research date:** 2026-09-02
**Ticket:** [GitHub issue #4](https://github.com/mattias-wiberg/saas-boilerplate/issues/4)
**Scope:** Deploy this repository from GitHub to local development and a Dokploy-managed production environment. This note covers source integration, build and runtime choices, configuration and secrets, health, operations, and recovery. No application code or CI configuration was changed.

The evidence below comes from current Dokploy documentation and official Next.js documentation. The repository links record the state inspected for this research.

## Recommended Decision Set

1. **Local development:** Run the app natively with `pnpm dev`, not in Docker. Use the ignored `.env.local` file for local-only values. Next.js specifically recommends local development over Docker on Mac and Windows, and this repository pins `pnpm` in `package.json`. [N-Local] [R-Package] [R-Gitignore]
2. **Production service:** Use a Dokploy **Application**, not the Static build type. The current app reads cookies and server-side Supabase data; Next.js static exports do not support cookies or request-dependent server features. [R-Page] [N-Static] [D-Applications]
3. **Production build path:** Prefer GitHub Actions to build a Docker image and publish it to GHCR or another private registry, then let Dokploy deploy that image. Dokploy's production guide calls CI/CD the recommended way to avoid resource-heavy builds on the production server. [D-GoingProd] [D-BuildServer] [D-GHCR]
4. **Initial/simple fallback:** A GitHub App-backed Dokploy Application using Nixpacks or Railpack is supported and is useful for a first deployment, but pin the Node major version explicitly. Nixpacks documents Node 18 as its default, while the current Next.js requirement is Node 20.9 or newer. [D-GitHub] [D-BuildType] [X-Nixpacks] [N-Install]
5. **Environment model:** Treat local development as a developer-machine concern. If a shared remote development target is later needed, create a separate Dokploy `development` environment or branch-specific Application; keep production variables and services isolated. [D-Multi] [D-GitHub]
6. **Configuration:** Set server-only values at Dokploy runtime. Set every `NEXT_PUBLIC_*` value that differs by target before `next build`, because Next.js freezes those values into browser JavaScript at build time. This repository currently uses `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY`. [N-Env] [R-SupabaseClient] [R-SupabaseServer]
7. **Availability and recovery:** Add a cheap `/health` endpoint and ensure `curl` exists in the runtime image before enabling Dokploy zero-downtime or automatic rollback settings. Use Swarm `start-first` updates with health checks; retain registry-based rollback for manual rollback to an identified image. [D-Zero] [D-Rollbacks] [N-Route]
8. **Trigger policy:** For an uncomplicated flow, select `main` in the GitHub-backed production Application and use GitHub auto-deploy. For a controlled production flow, require CI to pass and publish the image, then trigger Dokploy with its registry webhook or API. [D-GitHub] [D-AutoDeploy] [D-GoingProd]

## Repository Fit

- The repository already has the standard `dev`, `build`, and `start` scripts expected by a production Next.js server, and its package manager is pinned to `pnpm@10.33.0`. [R-Package] [N-Deploy]
- Next.js 16.3.4 requires Node.js 20.9 or newer. Any Dokploy builder or Docker base image must meet that requirement. [R-Package] [N-Install]
- The current page calls `cookies()` and queries Supabase from a server component, so this is a server-backed Next.js application rather than a static-only site. [R-Page] [N-Static]
- The current `next.config.ts` does not enable `output: "standalone"`. Standalone output is an optional Docker optimization, not a prerequisite for the normal `next start` path. [R-NextConfig] [N-Output]
- The current application tree has no `/health` route. A health endpoint is therefore a deployment prerequisite for the recommended Swarm health-check and rollback configuration, not an existing capability of this branch. [R-AppTree] [D-Rollbacks]

## Deployment Topology

### Local development

Use `pnpm dev` on the developer machine. `next dev` is a development server and is intentionally different from `next build` plus `next start`; Next.js recommends reserving Docker for production deployments and production-build testing when developing on Mac or Windows. [N-Install] [N-Local]

Keep local environment values in `.env.local`. The repository ignores `.env*`, and Next.js warns that `.env` files are almost never meant to be committed. [R-Gitignore] [N-Env]

### Dokploy project and environments

Dokploy organizes services as Organization -> Project -> Environment -> Service. Environments isolate services and provide environment-level variables; production and development should not share an environment. [D-Multi]

For this repository, the smallest useful setup is:

```text
Dokploy project: saas-boilerplate
  production environment
    Next.js Application: production
```

If a shared remote development or staging target becomes necessary, add a separate environment and Application, with its own domain and variables. Dokploy's GitHub guide explicitly describes separate Applications for development, staging, and production when they use different branches of the same repository. [D-Multi] [D-GitHub]

### GitHub source integration

The supported GitHub flow is:

1. Create a Dokploy GitHub App and authorize only the repository or repositories it needs.
2. Create an Application and select the repository, branch, and build path (`/` for this repository).
3. Set the Application's build and runtime configuration.
4. Deploy and attach a domain routed to the container port.

Dokploy says GitHub integration provides automatic deployments on pushes to the selected branch. A push to a different branch does not deploy that Application. [D-GitHub]

## Runtime and Build Choices

### What "Node runtime" means here

Next.js supports both a Node.js server and a Docker container, with all Next.js features available in either deployment form. A Node.js deployment requires a production build followed by a production start command. [N-Deploy]

Dokploy Applications are managed as services or containers, and its build types produce the container that Dokploy runs. Therefore, the practical "Node runtime" choice in Dokploy is `next start` inside a Nixpacks, Railpack, or Dockerfile-built container, not an unmanaged Node process on the host. [D-Applications] [D-BuildType]

### Direct GitHub build: Nixpacks or Railpack

Dokploy documents Nixpacks as its default build type and exposes install, build, and start overrides through `NIXPACKS_*` variables. Its newer Railpack build type exposes corresponding `RAILPACK_*` variables. [D-BuildType]

The repository shape is compatible with the Node providers: Nixpacks uses the `packageManager` field or lockfile to select the package manager and runs the `build` script; Railpack likewise detects the repository's package manager and uses the `start` script. [X-Nixpacks] [X-Railpack] [R-Package]

There is a version trap. Nixpacks documents Node 18 as its default major version, but Next.js 16.3.4 requires Node 20.9 or newer. If Nixpacks is used, set `NIXPACKS_NODE_VERSION=20` or a newer supported major and verify the selected version in the deployment log. If Railpack is used, set `RAILPACK_NODE_VERSION` or declare an explicit repository runtime version rather than relying on an implicit default. [X-Nixpacks] [X-Railpack] [N-Install]

This path is the fastest way to validate the application, but Dokploy's production guidance warns that building on the deployment server can consume enough CPU, RAM, and disk to cause timeouts or affect other services. [D-GoingProd]

### Explicit Dockerfile or prebuilt image

For a controlled production path, build the image in GitHub Actions and publish it to GHCR or another registry. Dokploy's documented production flow builds and pushes an image in CI, configures the Dokploy Application with a Docker image, and then deploys it. [D-GoingProd] [D-GHCR] [D-Docker]

Dokploy can also use a Dockerfile build type directly from the GitHub source. That gives control over the build context, Dockerfile path, and build stage, but the build still consumes resources on the configured Dokploy build host unless a separate build server is used. [D-BuildType] [D-BuildServer]

For a later Dockerfile implementation, Next.js `output: "standalone"` creates `.next/standalone` and a minimal `server.js` containing the traced runtime files. The Docker image must also copy `public` and `.next/static` when the application uses them. The standalone server accepts `PORT` and `HOSTNAME` environment variables. [N-Output]

The CI image should use an immutable commit or release identifier rather than `latest`; Dokploy's production hardening guidance warns against floating production image tags because a redeploy can otherwise pull an untested image. [D-Hardening]

### Static export is not the right choice

Next.js static export produces files for a static web server but does not support cookies, request-dependent Route Handlers, Proxy, Server Actions, ISR, or other server-required features. The current page's cookie access alone rules out selecting Dokploy's Static build type for the baseline application. [N-Static] [R-Page] [D-BuildType]

## Build and Start Contract

| Concern | Decision | Evidence |
| --- | --- | --- |
| Install | Use the repository's pinned pnpm toolchain and lockfile. | [R-Package] [X-Nixpacks] |
| Build | Run `pnpm build` / `pnpm run build`, which invokes `next build`. | [R-Package] [N-Deploy] |
| Start | Run `pnpm start` / `pnpm run start`, which invokes `next start`. | [R-Package] [N-Deploy] |
| Node version | Use Node 20.9 or newer; set the builder version explicitly. | [N-Install] [X-Nixpacks] [X-Railpack] |
| Container port | Use port 3000 unless `PORT` is deliberately overridden; route the Dokploy domain to the same internal port. | [N-CLI] [D-Domains] |
| Production mode | Run the production build before the production start command; never use `next dev` as the production process. | [N-Deploy] [N-CLI] |
| Linting | Run `pnpm lint` as a separate CI check. Next.js 16 no longer runs the linter automatically during `next build`. | [R-Package] [N-Install] |

`PORT` is a server-start setting, not a value to place in a Next.js `.env` file. Set it in the Dokploy service or container environment when overriding the default. [N-CLI]

## Environment and Secret Injection

### Dokploy variable scopes

Dokploy supports project-level shared variables, environment-level variables, and service-level variables. Service-level values can override broader scopes, and Dokploy supports references such as `${{project.NAME}}` and `${{environment.NAME}}`. [D-Variables] [D-Multi]

Recommended placement:

| Value type | Local | Dokploy |
| --- | --- | --- |
| Developer-only values | `.env.local` | Not needed |
| Shared non-secret configuration | Local `.env` defaults where appropriate | Project-level variable |
| Production/staging endpoint or credential | Target-specific local file or shell environment | Environment-level or service-level variable |
| Sensitive production secret | Local secret store | Dokploy Secrets Provider reference |
| Browser-visible `NEXT_PUBLIC_*` value | Local value before `next dev`/build | Value supplied before the target build |

Dokploy Secrets Providers can fetch secrets from supported external managers at deploy time. The reference is stored in the configuration, the value is not stored in the Dokploy database, and rotating a provider secret takes effect on the next deployment. A missing secret fails deployment rather than silently injecting an empty value. [D-Secrets]

For Dockerfile builds, Dokploy distinguishes build-time secrets from build arguments. Its guidance says not to pass secrets through build arguments or ordinary environment variables because they persist in the image or build history; use Docker build secrets for sensitive build inputs. [D-BuildType]

### Next.js build-time versus runtime values

Next.js keeps unprefixed environment variables on the server by default. Values prefixed `NEXT_PUBLIC_` are inlined into the JavaScript bundle during `next build` and do not change when only the running container environment changes. Server-side values can be read at runtime during dynamic rendering. [N-Env] [N-Self]

This repository reads `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` in both its browser helper and server helper. [R-SupabaseClient] [R-SupabaseServer]

Operational consequence: if local and production use different Supabase projects, the production build must receive the production `NEXT_PUBLIC_*` values before `next build`. A single image cannot safely be promoted between those targets and have those browser values change at container start. Keep genuinely private server credentials unprefixed and inject them at runtime. This is an application configuration constraint, not a Dokploy limitation. [N-Env] [N-Self]

## Health, Zero Downtime, and Rollback

### Health endpoint

Dokploy's zero-downtime and automatic rollback examples assume a `/health` endpoint returning HTTP 200 on port 3000. Its rollback guide also requires `curl` in the container, noting that Alpine images do not include it by default. [D-Zero] [D-Rollbacks]

The future health endpoint should be cheap and independent of the Supabase query used by the home page. A Next.js App Router Route Handler can return a `Response` for a `GET` request, which is sufficient for the endpoint shape Dokploy expects. [N-Route]

### Dokploy Swarm settings

Dokploy's documented example uses the following health check:

```json
{
  "Test": ["CMD", "curl", "-f", "http://localhost:3000/health"],
  "Interval": 30000000000,
  "Timeout": 10000000000,
  "StartPeriod": 30000000000,
  "Retries": 3
}
```

For the update configuration, the example uses one-at-a-time updates, a 10-second delay, rollback on failure, and `start-first` ordering:

```json
{
  "Parallelism": 1,
  "Delay": 10000000000,
  "FailureAction": "rollback",
  "Order": "start-first"
}
```

These are Dokploy's starting values, not an assertion that every application should use them unchanged. Validate the health route and startup time in a non-production deployment before relying on them. [D-Zero] [D-Rollbacks]

Without this configuration, Dokploy documents the default behavior as stopping the running container before starting the new one, which can produce a Bad Gateway while the new container initializes. [D-Zero]

### Rollback options

Dokploy documents two distinct mechanisms:

- **Automatic Swarm rollback:** The new service version fails its configured health check during an update, so Swarm rolls back to the previous version. It depends on both a valid health check and the container's `curl` binary. [D-Rollbacks]
- **Manual registry rollback:** Dokploy stores each deployment image in a configured registry, associates the image with the deployment record, and exposes a rollback action for any retained deployment. This requires a configured registry, credentials, and image push support. [D-Rollbacks] [D-Registry]

Use automatic rollback for failed startup/readiness and manual registry rollback for an intentional return to a known image. Do not treat a successful container start as proof that the application is healthy.

## Triggers, Redeploy, and Logs

### Trigger paths

- **GitHub App push:** The selected branch is automatically deployed by the GitHub integration. Branch selection is the gate; pushes to other branches do not deploy that Application. [D-GitHub]
- **Repository webhook:** Enable Auto Deploy in the Application, copy the deployment webhook URL, and configure the Git provider to call it. Dokploy documents GitHub, GitLab, Bitbucket, Gitea, and Docker Hub support, with Docker Hub limited to Applications. [D-AutoDeploy]
- **Dokploy API:** A CI job or script can call `POST /api/application.deploy` with an `x-api-key` after identifying the Application ID. Use a separately scoped, expiring token where the Dokploy version supports it. [D-AutoDeploy] [D-Hardening]
- **CI image flow:** GitHub Actions builds and publishes the image; a registry webhook or the Dokploy API triggers the deployment. This keeps build/test policy in GitHub and runtime orchestration in Dokploy. [D-GoingProd] [D-AutoDeploy]
- **Pull-request previews:** Dokploy can create and update preview deployments for GitHub pull requests, but its documentation warns against enabling previews for public repositories because external contributors can execute builds and deployments on the server. [D-Preview]

Recommended production trigger: merge to protected `main` only after CI passes, publish an image tagged with the commit identifier, and trigger the production Dokploy Application. Direct GitHub auto-deploy remains an acceptable early-stage fallback when a separate CI image pipeline is not yet justified. [D-GitHub] [D-GoingProd] [D-Hardening]

### Redeploy and deployment records

Dokploy's Application UI exposes deployments, logs, and a deployment record while a build is running. The current Applications guide says the UI shows the last 10 deployments; queued deployments can be canceled, while deployments already in progress cannot. [D-Applications]

Configuration changes such as cluster or resource settings require a redeploy to take effect. A registry rollback also requires waiting for the image pull before the container appears in the Logs tab. [D-Advanced] [D-Rollbacks]

Use the deployment record and build log to verify the selected branch, Node version, install command, build command, and start command. Never print secret values into those logs.

### Logs and monitoring ownership

Dokploy exposes per-Application logs and a monitoring view. Its Applications guide describes four resource graphs and says the values update while the page is being viewed; the separate Monitoring guide labels the advanced monitoring feature as Cloud-only. For self-hosted production, Dokploy's hardening guide recommends external log shipping and alerting rather than relying on the panel as the durable observability system. [D-Applications] [D-Monitoring] [D-Hardening]

Logs and monitoring are placement-sensitive. Dokploy documents that the UI cannot access logs or monitoring when an Application runs on a different worker node, and that remote-server log loading can also fail when the server is slow or short on disk. [D-TroubleshootingLogs]

## Operational Ownership

| Topology | What runs where | Operational consequence |
| --- | --- | --- |
| Dokploy Server | UI, builds, and Applications share one server. | Simplest starting point; the operator owns the server's capacity, Docker, Dokploy uptime, updates, backups, network, and logs. [D-DeploymentOptions] [D-CloudVs] |
| Remote deployment server | Dokploy manages an independent server over SSH; the server runs Docker, Traefik, and the Application. | Separates production workload from the control plane, but requires SSH access, root setup, bash, host security, disk, and Docker operations. [D-Remote] [D-RemoteDeploy] |
| Dedicated build server | Dokploy clones and builds on the build server, pushes to a registry, and the deployment server pulls the image. | Reduces production build pressure, but requires another server, SSH, Docker, registry, cleanup, and image lifecycle ownership. Build servers are for Applications only. [D-BuildServer] |
| Dokploy Cloud control plane | Dokploy manages the UI, database, and management layer; Applications still run on the operator's servers. | Removes control-plane uptime and update work, but the operator still owns workload servers, domains, runtime configuration, external secrets, and application operations. [D-Cloud] [D-CloudVs] |

For a small local-plus-production setup, start with native local development and one production Application on a deployment server. Use a separate remote production server or Dokploy Cloud when keeping the control plane away from production matters. Add a dedicated build server only if GitHub Actions is not the preferred build owner or build pressure justifies another host. [D-DeploymentOptions] [D-BuildServer] [D-GoingProd]

Self-hosted installation has a documented minimum of 2 GB RAM and 30 GB disk, and requires ports 80, 443, and 3000 to be available during installation. The hardening guide recommends keeping the Dokploy UI port 3000 closed to the public internet after putting the panel behind HTTPS or a VPN. [D-Install] [D-Hardening]

## Implementation Boundary

This ticket is research-only. The following are deliberate follow-up items, not changes made here:

1. Add and test a `/health` Route Handler that does not depend on the home page's database query.
2. Decide whether to use direct Dokploy GitHub builds first or add a Dockerfile and GitHub Actions image pipeline.
3. Choose production and local Supabase values, supplying production `NEXT_PUBLIC_*` values during the production build.
4. Configure Dokploy's production Application, port 3000 domain, environment variables or Secrets Provider references, health check, update policy, registry rollback, and external log/alert destination.

## Sources

Sources were accessed on 2026-09-02.

### Dokploy

- [D-Applications: Applications](https://docs.dokploy.com/docs/core/applications)
- [D-GitHub: GitHub integration](https://docs.dokploy.com/docs/core/github)
- [D-Providers: Deployment providers](https://docs.dokploy.com/docs/core/providers)
- [D-BuildType: Application build types](https://docs.dokploy.com/docs/core/applications/build-type)
- [D-Docker: Docker Registry source](https://docs.dokploy.com/docs/core/Docker)
- [D-GoingProd: Going Production](https://docs.dokploy.com/docs/core/applications/going-production)
- [D-AutoDeploy: Auto Deploy](https://docs.dokploy.com/docs/core/auto-deploy)
- [D-Variables: Environment Variables](https://docs.dokploy.com/docs/core/variables)
- [D-Secrets: Secrets Providers](https://docs.dokploy.com/docs/core/secrets-providers)
- [D-Domains: Domains](https://docs.dokploy.com/docs/core/domains)
- [D-Advanced: Application Advanced settings](https://docs.dokploy.com/docs/core/applications/advanced)
- [D-Zero: Zero Downtime](https://docs.dokploy.com/docs/core/applications/zero-downtime)
- [D-Rollbacks: Rollbacks](https://docs.dokploy.com/docs/core/applications/rollbacks)
- [D-Preview: Preview Deployments](https://docs.dokploy.com/docs/core/applications/preview-deployments)
- [D-Multi: Multi-Tenancy](https://docs.dokploy.com/docs/core/multi-tenancy)
- [D-DeploymentOptions: Deployment Options](https://docs.dokploy.com/docs/core/deployment-options)
- [D-Remote: Remote Servers](https://docs.dokploy.com/docs/core/remote-servers)
- [D-RemoteDeploy: Remote Server Deployments](https://docs.dokploy.com/docs/core/remote-servers/deployments)
- [D-BuildServer: Build Server](https://docs.dokploy.com/docs/core/remote-servers/build-server)
- [D-Cloud: Dokploy Cloud](https://docs.dokploy.com/docs/core/cloud)
- [D-CloudVs: Cloud vs Self-Hosted](https://docs.dokploy.com/docs/core/differences)
- [D-Install: Installation](https://docs.dokploy.com/docs/core/installation)
- [D-GHCR: GitHub Container Registry](https://docs.dokploy.com/docs/core/registry/ghcr)
- [D-Registry: Registry](https://docs.dokploy.com/docs/core/registry)
- [D-Monitoring: Monitoring](https://docs.dokploy.com/docs/core/monitoring)
- [D-TroubleshootingLogs: Logs and Monitoring troubleshooting](https://docs.dokploy.com/docs/core/troubleshooting/logs-monitoring)
- [D-Hardening: Production Hardening Guide](https://docs.dokploy.com/docs/core/guides/production-hardening)

### Next.js and build providers

- [N-Deploy: Next.js Deploying](https://nextjs.org/docs/app/getting-started/deploying)
- [N-Install: Next.js Installation and system requirements](https://nextjs.org/docs/app/getting-started/installation)
- [N-Local: Next.js Local Development](https://nextjs.org/docs/app/guides/local-development)
- [N-CLI: Next.js CLI](https://nextjs.org/docs/app/api-reference/cli/next)
- [N-Env: Next.js Environment Variables](https://nextjs.org/docs/app/guides/environment-variables)
- [N-Self: Next.js Self-Hosting](https://nextjs.org/docs/app/guides/self-hosting)
- [N-Static: Next.js Static Exports](https://nextjs.org/docs/app/guides/static-exports)
- [N-Output: Next.js Output File Tracing](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)
- [N-Route: Next.js Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route)
- [X-Nixpacks: Nixpacks Node provider](https://nixpacks.com/docs/providers/node)
- [X-Railpack: Railpack Node provider](https://railpack.com/languages/node)

### Repository state

- [R-Package: package.json](https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/package.json)
- [R-Gitignore: .gitignore](https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/.gitignore)
- [R-Page: app/page.tsx](https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/app/page.tsx)
- [R-SupabaseClient: browser Supabase helper](https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/utils/supabase/client.ts)
- [R-SupabaseServer: server Supabase helper](https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/utils/supabase/server.ts)
- [R-NextConfig: next.config.ts](https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/next.config.ts)
- [R-AppTree: current app directory](https://github.com/mattias-wiberg/saas-boilerplate/tree/research/dokploy-nextjs-deployment-path/app)

[D-Applications]: https://docs.dokploy.com/docs/core/applications
[D-GitHub]: https://docs.dokploy.com/docs/core/github
[D-Providers]: https://docs.dokploy.com/docs/core/providers
[D-BuildType]: https://docs.dokploy.com/docs/core/applications/build-type
[D-Docker]: https://docs.dokploy.com/docs/core/Docker
[D-GoingProd]: https://docs.dokploy.com/docs/core/applications/going-production
[D-AutoDeploy]: https://docs.dokploy.com/docs/core/auto-deploy
[D-Variables]: https://docs.dokploy.com/docs/core/variables
[D-Secrets]: https://docs.dokploy.com/docs/core/secrets-providers
[D-Domains]: https://docs.dokploy.com/docs/core/domains
[D-Advanced]: https://docs.dokploy.com/docs/core/applications/advanced
[D-Zero]: https://docs.dokploy.com/docs/core/applications/zero-downtime
[D-Rollbacks]: https://docs.dokploy.com/docs/core/applications/rollbacks
[D-Preview]: https://docs.dokploy.com/docs/core/applications/preview-deployments
[D-Multi]: https://docs.dokploy.com/docs/core/multi-tenancy
[D-DeploymentOptions]: https://docs.dokploy.com/docs/core/deployment-options
[D-Remote]: https://docs.dokploy.com/docs/core/remote-servers
[D-RemoteDeploy]: https://docs.dokploy.com/docs/core/remote-servers/deployments
[D-BuildServer]: https://docs.dokploy.com/docs/core/remote-servers/build-server
[D-Cloud]: https://docs.dokploy.com/docs/core/cloud
[D-CloudVs]: https://docs.dokploy.com/docs/core/differences
[D-Install]: https://docs.dokploy.com/docs/core/installation
[D-GHCR]: https://docs.dokploy.com/docs/core/registry/ghcr
[D-Registry]: https://docs.dokploy.com/docs/core/registry
[D-Monitoring]: https://docs.dokploy.com/docs/core/monitoring
[D-TroubleshootingLogs]: https://docs.dokploy.com/docs/core/troubleshooting/logs-monitoring
[D-Hardening]: https://docs.dokploy.com/docs/core/guides/production-hardening
[N-Deploy]: https://nextjs.org/docs/app/getting-started/deploying
[N-Install]: https://nextjs.org/docs/app/getting-started/installation
[N-Local]: https://nextjs.org/docs/app/guides/local-development
[N-CLI]: https://nextjs.org/docs/app/api-reference/cli/next
[N-Env]: https://nextjs.org/docs/app/guides/environment-variables
[N-Self]: https://nextjs.org/docs/app/guides/self-hosting
[N-Static]: https://nextjs.org/docs/app/guides/static-exports
[N-Output]: https://nextjs.org/docs/app/api-reference/config/next-config-js/output
[N-Route]: https://nextjs.org/docs/app/api-reference/file-conventions/route
[X-Nixpacks]: https://nixpacks.com/docs/providers/node
[X-Railpack]: https://railpack.com/languages/node
[R-Package]: https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/package.json
[R-Gitignore]: https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/.gitignore
[R-Page]: https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/app/page.tsx
[R-SupabaseClient]: https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/utils/supabase/client.ts
[R-SupabaseServer]: https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/utils/supabase/server.ts
[R-NextConfig]: https://github.com/mattias-wiberg/saas-boilerplate/blob/research/dokploy-nextjs-deployment-path/next.config.ts
[R-AppTree]: https://github.com/mattias-wiberg/saas-boilerplate/tree/research/dokploy-nextjs-deployment-path/app
