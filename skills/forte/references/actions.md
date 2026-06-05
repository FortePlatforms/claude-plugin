# Actions (beta)

Forte Actions call one of your Forte **services** asynchronously, at times you choose — either on a
recurring schedule or once at a specific moment. Like a [payment trigger](payments.md), an action targets
a **service + path** in the same project (not an arbitrary external URL); Forte sends an HTTP `POST` to that
service over the private network and records each call as an **invocation**. Actions are managed from your
backend (`forte.projects.*` with `FORTE_API_TOKEN`), the `forte actions` CLI, or the console — they belong
to the project, not to a signed-in user.

## Schedule types

- **Recurring** — a cron expression evaluated in an IANA timezone (e.g. `America/New_York`), with an
  optional active window (start/end). An action fires **at most once per minute**; sub-minute schedules
  are rejected (`ACTION_SCHEDULE_TOO_FREQUENT`).
- **One-time** — a single future timestamp. The action fires once, then stops.

A project has a limited number of enabled actions (`ACTION_QUOTA_EXCEEDED` past the limit). Pause an action
(`enabled: false`) to free a slot without deleting it. The target service must exist in the project
(`ACTION_TARGET_SERVICE_NOT_FOUND` otherwise).

## The request Forte sends

Each invocation is an HTTP `POST` to `targetPath` on `targetServiceId`, with the body you configured. The
receiving service sees these headers:

- `X-Forte-Action-Id` — the action that fired.
- `X-Forte-Action-Invocation-Id` — this specific invocation (use it to dedupe).
- `X-Forte-Trusted: 1` — the same trusted-request signal used for payment triggers. **Validate this
  header**: Forte strips inbound `X-Forte-*`, so its presence proves the call came from Forte.

Because the target is a Forte service, the request is already authenticated as Forte-internal — no public
URL is exposed. Make handlers **idempotent on the invocation id** and return `2xx` quickly.

## Retries

Mark an action **retryable** and Forte retries a failed delivery (a non-`2xx` response, timeout, or
connection error) on its internal policy. A non-retryable action is attempted once. Either way, the
service failing is recorded on the invocation — it does not count against Forte.

## Invocations

Each execution is recorded as an **invocation** you can list and inspect (status, attempts, response code,
timing). Statuses include `SUCCEEDED`, `FAILED` (the service errored after retries), `PENDING`/`RUNNING`/
`RETRYING`, `CANCELLED` (cancelled before it ran), and `MISSED` (a scheduled occurrence Forte did not run
on time).

- **Invoke now** — create an on-demand invocation to test your service immediately.
- **Cancel** — cancel a still-`PENDING` invocation.
- **Pause / resume** — toggle `enabled` to stop/start the schedule without losing the action.

## CLI

```bash
# Recurring: 9:00 AM every day, New York time, retried on failure
forte actions create <projectId> \
  --name nightly-report --service svc_<id> --path /jobs/nightly \
  --schedule recurring --cron "0 9 * * *" --timezone America/New_York --retryable

# One-time: run once at a specific instant
forte actions create <projectId> \
  --name launch-ping --service svc_<id> --path /hooks/launch \
  --schedule one-time --at 2026-07-01T16:00:00Z

forte actions list <projectId>
forte actions get <projectId> <actionId>
forte actions invoke <projectId> <actionId>            # run once, now
forte actions invocations <projectId> <actionId>       # delivery history
forte actions update <projectId> <actionId> --disable  # pause
forte actions cancel <projectId> <actionId> <invocationId>
```

Omit an id and the CLI prompts you to pick one.

## SDK / API (server-side only)

```typescript
import { ForteClient } from "@forteplatforms/sdk";

const forte = new ForteClient({ apiToken: process.env.FORTE_API_TOKEN });

const action = await forte.projects.createAction({
  projectId,
  createActionRequest: {
    name: "nightly-report",
    targetServiceId: "svc_...",          // a service in this project
    targetPath: "/jobs/nightly",
    requestBody: JSON.stringify({ job: "nightly-report" }),
    scheduleType: "RECURRING",
    cronExpression: "0 9 * * *",
    timezone: "America/New_York",
    retryable: true,
  },
});

await forte.projects.createActionInvocation({ projectId, actionId: action.actionId }); // invoke now
const { items } = await forte.projects.listActionInvocations({ projectId, actionId: action.actionId });
```

Handler on the target service (verify the trusted header, dedupe on the invocation id, return 2xx fast):

```typescript
app.post("/jobs/nightly", (req, res) => {
  if (req.header("X-Forte-Trusted") !== "1") return res.sendStatus(401);
  const invocationId = req.header("X-Forte-Action-Invocation-Id");
  if (alreadyProcessed(invocationId)) return res.sendStatus(200);
  enqueueWork(req.body);
  res.sendStatus(200);
});
```

Canonical docs: [forteplatforms.com/docs/core-concepts/actions](https://forteplatforms.com/docs/core-concepts/actions)
and [forteplatforms.com/docs/guides/creating-actions](https://forteplatforms.com/docs/guides/creating-actions)
