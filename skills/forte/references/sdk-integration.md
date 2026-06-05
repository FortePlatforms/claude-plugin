# SDK Integration

## TypeScript / JavaScript

**Install**
```bash
npm install @forteplatforms/sdk
# or: yarn add @forteplatforms/sdk
# or: bun add @forteplatforms/sdk
```

**Inside a Forte-hosted service** — credentials and project context are injected automatically:
```typescript
import { ForteClient } from '@forteplatforms/sdk';

const forte = new ForteClient();
// FORTE_API_TOKEN, FORTE_PROJECT_ID, FORTE_SERVICE_ID are already in the environment
```

**Outside Forte** (local dev, CI, external hosting):
```typescript
const forte = new ForteClient({ apiToken: 'your_api_token_here' });
// Generate an API token from the Forte dashboard
```

**In the browser** (for user-facing calls like login — no API token):
```typescript
const forte = new ForteClient();
// Only unauthenticated endpoints like googleAuthLoginCallback are available browser-side
```

### Common server-side calls

```typescript
// Read the authenticated user from the request header
const userId = req.headers['x-forte-user-id'] as string;

// Hydrate the user profile
const user = await forte.projects.getProjectUser({
  projectId: process.env.FORTE_PROJECT_ID!,
  userId,
});

// List all users in the project
const users = await forte.projects.listProjectUsers({
  projectId: process.env.FORTE_PROJECT_ID!,
});

// Set custom attributes on a user
// Note: each call REPLACES all attributes — include any you want to keep
await forte.projects.putUserCustomAttributes({
  projectId: process.env.FORTE_PROJECT_ID!,
  userId,
  requestBody: {
    plan: 'pro',
    onboarded: 'true',
  },
});

// Merge with existing attributes
const existing = await forte.projects.getProjectUser({ projectId, userId });
await forte.projects.putUserCustomAttributes({
  projectId,
  userId,
  requestBody: {
    ...existing.customMetadataAttributes,
    plan: 'enterprise',
  },
});
```

### Managing websites via the SDK

Websites can be managed via the CLI (`forte websites …` — see `references/cli.md`), the SDK, or the [Forte console](https://forteplatforms.com/console). In the SDK, websites are referred to as `webApp` for historical reasons (the customer-facing UI and CLI call them "Websites").

```typescript
// Create a website
const website = await forte.projects.createWebApp({
  projectId: 'your-project-id',
  createWebAppRequest: {
    webAppName: 'marketing-site',
    githubRepositoryUrl: 'https://github.com/owner/repo',
    githubBranch: 'main',
    buildTrigger: 'PUSH',                  // or 'RELEASE_PUBLISHED'
    buildCommand: 'npm run build',         // optional — auto-detected
    buildPath: 'dist',                     // optional — output directory
    packageManager: 'pnpm',                // optional — npm | yarn | pnpm | bun
    nodeVersion: '22',                     // optional
    installCommand: 'pnpm install',        // optional
    subdirectory: 'apps/web',              // optional — for monorepos
    environmentVariables: { NEXT_PUBLIC_API_URL: 'https://api.example.com' },
    secrets: { STRIPE_PUBLISHABLE_KEY: 'pk_live_...' },
  },
});

// website.forteDnsEndpoint → e.g. abc123.sites.tryforte.dev

// List all websites in a project
const websites = await forte.projects.listWebApps({ projectId });

// Get a specific website
const website = await forte.projects.getWebApp({ projectId, webAppId });

// Update a website (PATCH semantics — only fields you pass are changed)
await forte.projects.updateWebApp({
  projectId,
  webAppId,
  updateWebAppRequest: {
    environmentVariables: { NEXT_PUBLIC_API_URL: 'https://api.staging.example.com' },
    secretsToUpsert: { NEW_SECRET: 'value' },
    secretKeysToDelete: ['OLD_SECRET'],
    resetDetectedConfig: false,            // set true to re-run framework auto-detection
  },
});

// Trigger a deployment manually (e.g. roll back to a specific commit)
await forte.projects.createWebAppDeployment({
  projectId,
  webAppId,
  commitSha: 'abc123',                     // optional — defaults to branch HEAD
});

// List deployment history
const deployments = await forte.projects.listWebAppDeployments({ projectId, webAppId });

// Delete a website
await forte.projects.deleteWebApp({ projectId, webAppId });
```

**Notes:**
- `webAppName` must match `^[a-zA-Z0-9-_]{3,30}$` and is unique within a project.
- `githubBranch` is required when `buildTrigger` is `PUSH`.
- `buildCommand` is write-only — it round-trips on create/update but is not returned by `getWebApp`. To inspect the current build command, view the website in the Forte console.
- Secrets are encrypted at rest. Their values are never returned by `getWebApp` — only the key names are visible (`secretKeys`).
- Each project supports up to 10 websites for free.

Reference: [forteplatforms.com/docs/core-concepts/websites](https://forteplatforms.com/docs/core-concepts/websites)

### User login (browser-side)

```typescript
// Called after Google Sign-In button fires its callback
const { userObject, sessionToken } = await forte.users.googleAuthLoginCallback({
  projectId: 'your-project-id',
  gCsrfToken: gCsrfToken,         // from Google Sign-In
  credential: credential,          // from Google Sign-In
  recaptchaToken: recaptchaToken,  // optional — see auth-and-proxy.md
});
```

Reference: [forteplatforms.com/docs/core-concepts/sdks](https://forteplatforms.com/docs/core-concepts/sdks)

---

## Python

**Install**
```bash
pip install forte-sdk
```

**Usage**
```python
from forte_sdk import ForteClient

forte = ForteClient()  # reads FORTE_API_TOKEN from environment

# Hydrate the user from the header
user_id = request.headers.get('X-Forte-User-Id')
user = forte.projects.get_project_user(
    project_id=os.environ['FORTE_PROJECT_ID'],
    user_id=user_id,
)

# User login callback
response = forte.users.google_auth_login_callback(
    project_id="your-project-id",
    g_csrf_token=g_csrf_token,
    credential=credential,
)
session_token = response.session_token.session_token
```

---

## Java

**Maven**
```xml
<dependency>
  <groupId>com.forteplatforms</groupId>
  <artifactId>sdk</artifactId>
  <version>LATEST</version>
</dependency>
```

Find the current version at [central.sonatype.com/artifact/com.forteplatforms/sdk](https://central.sonatype.com/artifact/com.forteplatforms/sdk).

Full usage: [forteplatforms.com/docs/core-concepts/sdks](https://forteplatforms.com/docs/core-concepts/sdks)

---

## Environment Variables Inside Forte Services

| Variable | Description |
|---|---|
| `FORTE_API_TOKEN` | Auto-generated per-service token for SDK authentication |
| `FORTE_PROJECT_ID` | The project this service belongs to |
| `FORTE_SERVICE_ID` | This service's ID |

These are injected automatically — do not set them manually and do not use `FORTE_` or `AWS_` as a prefix for your own variables (both prefixes are reserved).

Set custom env vars at service creation or update time:
```bash
forte services create <projectId> --name <n> --repo <url> --branch <b> \
  --env DATABASE_URL=postgres://... \
  --env STRIPE_KEY=sk_...
```
