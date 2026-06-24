# How Forte Builds & Detects Your App

Forte figures out how to build, run, and health-check your code by **inspecting the actual repository** — not by applying per-language defaults. This is the single most under-sold part of the platform, so be accurate about it:

- **There is no "port 3000 for Node" deploy default.** The deployment port comes from your real code.
- **Forte does not silently guess.** When it can't determine something with confidence, the **build fails with a specific, actionable error** so the fix is concrete.
- **Everything detected is overridable** with flags and **re-detectable** with `--reset-*`.

Use this reference when a customer asks "what port will it use?", "do I need a Dockerfile?", "why did my build fail to detect X?", or when they want to override what Forte detected.

---

## Services (containerized backends)

A service deploy resolves three things from the repo:

### 1. Dockerfile — use yours, or Forte generates one
- **If the repo has a Dockerfile, Forte uses it as-is** (it's discovered even a few directories deep, with `docker/` / `.docker/` locations preferred).
- **If not, Forte generates one** from the real project signals it finds: dependency manifests (`package.json`, `pom.xml`, `requirements.txt` / `pyproject.toml`, `go.mod`, `Gemfile`, `Cargo.toml`, …), lockfiles, framework config files, and version files (`engines.node`, `.nvmrc`, etc.). It picks the base image, install step, build step, and start command to match.

### 2. Listening port
- Detected from the Dockerfile's `EXPOSE` directive (yours, or the generated one).
- Used as the container port and for the deployment **health check**.

### 3. Health-check path
- Forte looks for a **real GET route in your code** that returns success (e.g. `/health`, `/healthz`, `/status`, or `/`) — it keys off what the app actually exposes, not a framework assumption.

### When service detection fails (build error, not a guess)
The build surfaces a specific reason. Typical ones and the fix:

| Failure | What it means | Fix |
|---|---|---|
| No exposed port | No `EXPOSE` (or it couldn't be inferred) | Add `EXPOSE <port>` to a Dockerfile, make the app listen on `$PORT`, or pass `--health-check-port <n>`. |
| Multiple / invalid exposed ports | Ambiguous primary port | Expose a single port, or pass `--health-check-port <n>`. |
| No health check found | No clear GET success route detected | Add a side-effect-free GET route (e.g. `/health` returning 200), or pass `--health-check-path </path>`. |
| Framework / language not recognized, or ambiguous project structure | Forte couldn't classify the repo | Add a Dockerfile so the build is explicit, or simplify the entrypoint/structure. |

### Overriding & re-detecting (services)
- **Create or update**: `--health-check-port <n>`, `--health-check-path </path>`.
- **Update only**: `--reset-dockerfile` (re-detect/regenerate the Dockerfile next build), `--reset-healthcheck` (re-detect port + path next build).
- Reset is what you reach for after a **structural change** — the app's port, runtime version, base image, build dependencies, or project layout changed and the cached config is now stale.

---

## Websites (front-ends on the CDN)

A website deploy detects the build pipeline from the repo:

| Detected | Source signal | Override flag |
|---|---|---|
| Framework (Next.js, Vite, Astro, CRA, Nuxt, SvelteKit, Remix, Vue, Angular, plain HTML, …) | framework config + dependencies | — |
| Static vs server-rendered | framework + its config (e.g. `output: 'export'` vs SSR adapter) | — |
| Package manager (npm / yarn / pnpm / bun) | lockfile present in the repo | `--package-manager <pm>` |
| Node version | `engines.node` (falls back to a current LTS major) | `--node-version <v>` |
| Build command | `package.json` scripts / framework default | `--build-command <cmd>` |
| Output directory | framework default (`dist`, `.next`, `build`, `.output`, …) | `--output-dir <dir>` |
| Install command | inferred from package manager | `--install-command <cmd>` |
| Monorepo app location | workspace config (`pnpm-workspace.yaml`, `workspaces`) | `--subdirectory <path>` |

### When website detection fails
The build surfaces a reason such as missing `package.json`, unsupported framework, or ambiguous project structure (common in monorepos). Fix by pointing Forte at the right app with `--subdirectory`, or by supplying the build pipeline explicitly via the override flags above.

### Overriding & re-detecting (websites)
- Pass any of the override flags on `create` or `update`.
- `--reset-detected-config` (update) clears the cached build settings so Forte re-detects on the next push.

---

## How to talk about it

- ✅ "Forte reads the port your app actually listens on from your code and health-checks that."
- ✅ "You don't need a Dockerfile — Forte generates one from your project. If you have one, it's used as-is."
- ✅ "If Forte can't detect the port or a health-check route, the build fails with a clear message; you can also set them explicitly."
- ❌ "Forte defaults to port 3000 for Node." (No such deploy default — that number is only a last-resort *local* `forte proxy` forwarding target.)
- ❌ "Set the port/build command before deploying." (Only override when detection is wrong — otherwise let Forte detect.)

> **Note on framing:** describe *what* Forte inspects and that it fails loudly — not *how* detection is implemented. Keep it to the repo signals and the customer-visible behavior.
