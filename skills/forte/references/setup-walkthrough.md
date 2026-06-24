# Setup Walkthrough

When the customer says "set up Forte for my app", "deploy this on Forte", "help me connect Forte", or similar, follow these steps. Use AskUserQuestion to collect decisions before running commands.

---

## Step 1 — Check CLI installation

Run `forte --version`. If not found, walk through installation from `references/cli.md` (Homebrew, curl, or manual binary) before continuing.

## Step 2 — Check authentication

Run `forte whoami`. If not authenticated, run `forte login` and wait for the browser flow to complete.

## Step 3 — Inspect the local repo (read-only)

Gather context without prompting the customer:

- **Git remote URL**: read `.git/config` or run `git remote get-url origin`
- **Current branch**: run `git rev-parse --abbrev-ref HEAD`
- **Framework / language**: check for `package.json` (Node.js/TypeScript), `requirements.txt` / `pyproject.toml` (Python), `pom.xml` / `build.gradle` (Java), etc. This is only to inform the **service-vs-website** decision below and the conversation — you don't need to get it exactly right, because Forte runs its own authoritative detection server-side at build time (see "How Forte builds your app" below).
- **Service or website?** — see the decision rules below
- **Already on Forte?**: run `forte projects list` to check for existing projects.

> **Don't supply or guess deployment build settings.** Forte auto-detects the listening port, health-check path, build/start commands, package manager, runtime version, and Dockerfile from the real code at build time — there is **no** "3000 for Node" deploy default. Only pass the override flags (`--health-check-port`, `--build-command`, `--output-dir`, etc.) when the customer explicitly wants to override what Forte detects. See "How Forte builds your app" below and `references/build-and-detection.md`.

> The only place a `3000`/`8000`/`8080` heuristic applies is choosing where **`forte proxy`** forwards locally (your local dev server's port): pass `-p <port>` if your dev server isn't on the service's detected port. That is a *local* concern, unrelated to how Forte health-checks the deployed service.

### Service vs. Website decision

Forte has two deployable primitives. Pick the right one based on what the repo contains:

| Repo signals | Resource type |
|---|---|
| Express / Fastify / Hono / NestJS / Django / Flask / FastAPI / Spring / Rails / any HTTP server | **Service** |
| Next.js with `output: 'export'`, Vite SPA, Astro with `output: 'static'`, Create React App, Vue/Angular static, plain `index.html` | **Website** |
| Next.js / Nuxt / SvelteKit running with SSR (no static export) | **Either** — a **Website** for simple public hosting, or a **Service** when you need Forte to authenticate and observe every request (see note below) |
| A monorepo with both a front-end and a back-end | **Both** — create one website AND one service, each with the appropriate `--subdirectory` (website) or `--base-directory` (service). See [Monorepo guide](https://forteplatforms.com/docs/guides/monorepo). |

If the answer is ambiguous, ask the customer.

**SSR websites bypass the Forte gateway.** A website — static *or* server-rendered — is served directly and does not pass through the gateway, so it gets **no** Forte auth enforcement, **no** `X-Forte-User-Id` header, and **no** request logs or metrics. Deploy an SSR app as a **service** when it needs Forte to authenticate or observe requests; deploy it as a **website** when it's public-by-default and you just want it hosted. See `references/auth-and-proxy.md` and [forteplatforms.com/docs/core-concepts/websites](https://forteplatforms.com/docs/core-concepts/websites).

**Websites are frontend only — pair them with a service for anything privileged.** A website cannot hold `FORTE_API_TOKEN` (never injected), cannot read `X-Forte-User-Id`, and has no request logs/metrics. So any code that needs the server-side Forte API (`forte.projects.*`), authenticated request handling, or observability must live in a **service**. The recommended shape for an app with a real backend is a **website (frontend) + a service (backend)** the browser calls with the user's session token as a Bearer header. The repo's `examples/nextjs-node/` shows exactly this (a Next.js website + a Node/Express service) and its README explains why the backend can't live in the website. If you'd rather keep backend logic inside one Next.js codebase, deploy that Next.js app as a **service** (not a website) so it runs behind the gateway with full tooling.

### How Forte builds your app

Forte figures out how to build and run your code from the **actual repository** — you don't hand it a port, a start command, or (usually) a Dockerfile:

- **Services**: Forte uses your repo's Dockerfile if it has one, otherwise it generates one from the real project signals (`package.json`/`pom.xml`/`requirements.txt`/`go.mod`/`Gemfile`/`Cargo.toml`, lockfiles, framework config, version files). It then detects the **listening port** (from the Dockerfile `EXPOSE`) and a **GET health-check route** (from the app's real routes) for the deployment health check.
- **Websites**: Forte detects the framework, package manager (from the lockfile), Node version (`engines.node`), build command, and output directory.

If Forte can't determine something with confidence, the **build fails with a specific error** (e.g. no exposed port, no health-check route found) instead of guessing — so the fix is concrete. Everything detected is **overridable** with flags and re-detectable with `--reset-*`. So when a customer asks "what port will it use?" the answer is *"the one your app actually listens on — Forte reads it from your code,"* not a per-language default. Full detail (signals, failure modes, override/reset flags): `references/build-and-detection.md`.

## Step 4 — Collect decisions

Ask these questions using AskUserQuestion BEFORE running any `forte` commands:

1. **Environments**: "Do you want one Forte project, or separate staging and production projects? Forte recommends two — they're free and unlimited." If they want a safe test environment (Stripe test-mode payments, hard-deletable users, contact-method overrides), mention **sandbox (test) mode** — but note it can only be enabled **at project creation time in the Console** (not via the CLI) and is **permanent** afterward. See `references/api-surfaces.md`.
2. **Service name**: "What should the service be called?" (suggest the repo name or package name from `package.json`)
3. **Branch**: "Which branch should trigger auto-deployment?" (suggest the current branch)
4. **Environment variables**: "Any environment variables to set now? Provide them as KEY=VAL pairs, or skip." (Note: `FORTE_` and `AWS_` prefixes are reserved — do not use them for custom vars.)
5. **Auth exclusions** (optional): "Are there routes that should bypass user auth, like `/health` or `/api/webhooks/**`?" (Ant patterns; skip if they don't know yet — configurable later with `forte services update`.)

## Step 5 — Create project(s)

**One project**:
```bash
forte projects create <name>
```

**Two projects (staging + production)**:
```bash
forte projects create <name>-staging
forte projects create <name>-production
```

Record the returned `projectId`(s) — used in the next step and later for `forte proxy`.

## Step 6 — Create the deployable resource(s)

### If creating a service:

```bash
forte services create <projectId> \
  --name <serviceName> \
  --repo <githubHttpsUrl> \
  --branch <branch> \
  [--env KEY=VAL ...] \
  [--secret KEY=VAL ...]
```

Don't pass port/health-check/Dockerfile flags here — Forte detects those from the code (see "How Forte builds your app" above). **Auth exclusions are set after creation** with `forte services update <projectId> <serviceId> --auth-exclude <antPattern> ...` (the `--auth-exclude` flag is not read on `create`).

If the GitHub App is not installed, an error will mention repository access — remind the customer to install it at [github.com/apps/forte-platforms](https://github.com/apps/forte-platforms), then re-run.

### If creating a website:

**Option A — CLI** (fastest for most developers):
```bash
forte websites create <projectId> \
  --name <name> \
  --repo <githubHttpsUrl> \
  --branch <branch> \
  [--build-command "npm run build"] \
  [--output-dir dist] \
  [--env KEY=VAL ...]
```
Forte auto-detects the framework. If the GitHub App is not installed, remind the customer to install it at [github.com/apps/forte-platforms](https://github.com/apps/forte-platforms).

**Option B — Forte console** (better for first-timers or when using the GitHub repo picker):
1. Open `https://forteplatforms.com/console/projects/<projectId>/websites/new`
2. Enter the website name, GitHub repository, build trigger (push or release), branch, and any advanced overrides.
3. Click **Create Website** — the first build starts automatically.

**Option C — Forte SDK** (when the customer is already calling other Forte SDK methods or wants this in CI):
```typescript
import { ForteClient } from '@forteplatforms/sdk';
const forte = new ForteClient({ apiToken: '<token>' });

const website = await forte.projects.createWebApp({
  projectId: '<projectId>',
  createWebAppRequest: {
    webAppName: 'marketing-site',
    githubRepositoryUrl: 'https://github.com/owner/repo',
    githubBranch: 'main',
    buildTrigger: 'PUSH',
    // Optional overrides — Forte auto-detects most popular frameworks
    // buildCommand: 'npm run build',
    // buildPath: 'dist',
    // packageManager: 'pnpm',
    // nodeVersion: '22',
    // subdirectory: 'apps/web',
    // environmentVariables: { NEXT_PUBLIC_API_URL: 'https://api.example.com' },
    // secrets: { STRIPE_PUBLISHABLE_KEY: 'pk_live_...' },
  },
});
// website.forteDnsEndpoint → e.g. abc123.sites.tryforte.dev
```

Once created, the website is reachable at `https://<websiteId>.sites.tryforte.dev`. To attach a custom domain, use `forte websites domains add <projectId> <websiteId> <domain>`, then `forte websites domains sync ...` to advance DNS/cert verification (see `references/cli.md`).

## Step 7 — Surface the results

For each project, service, and website created, show the Forte console URLs:

```
Project:  https://forteplatforms.com/console/projects/<projectId>
Service:  https://forteplatforms.com/console/projects/<projectId>/services/<serviceId>
Website:  https://forteplatforms.com/console/websites/<websiteId>
```

For websites, also show the public URL:
```
https://<websiteId>.sites.tryforte.dev
```

Do NOT write these to any file in the repo (no `FORTE.md`, no README edits) without the customer's explicit request.

**If Claude auto-memory is active for this repo** (a `.claude/projects/.../memory/MEMORY.md` exists), save the project ID(s) as a project memory entry so future Claude sessions can recall them:

```markdown
---
name: Forte project IDs
description: Forte project IDs for this repo, used to run CLI commands without looking them up each time
type: project
---

Staging project ID: <projectId>
Production project ID: <projectId>

Console: https://forteplatforms.com/console/projects/<projectId>
```

Write this as a new file (e.g., `memory/forte_project_ids.md`) and add a pointer to `MEMORY.md`.

## Step 8 — Offer next steps

After setup completes:

1. **SDK integration**: "Add the Forte SDK to your app?" → read `references/sdk-integration.md`
2. **Local testing** (services only): "Set up `forte proxy` so you can test real Forte auth against local code?" → read `references/auth-and-proxy.md`
3. **Google OAuth for users** (services only): "Configure your Google OAuth client ID?" → `forte projects update <projectId> --google-oauth-client-id <clientId>` — see [forteplatforms.com/docs/users/authentication](https://forteplatforms.com/docs/users/authentication)
4. **Custom build configuration** (websites): If Forte's framework auto-detection picks the wrong build command or output directory, the customer can override it from the website's Edit page in the console — see [forteplatforms.com/docs/core-concepts/websites](https://forteplatforms.com/docs/core-concepts/websites).
