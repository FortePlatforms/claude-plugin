# Actions (beta)

Forte Actions call one of your Forte **services** asynchronously, at times you choose — either on a
recurring schedule or once at a specific moment. Like a [payment trigger](payments.md), an action targets
a **service + path** in the same project (not an arbitrary external URL); Forte sends an HTTP `POST` to that
service over the private network and records each call as an **invocation**. Actions are managed from your
backend (`forte.projects.*` with `FORTE_API_TOKEN`), the `forte actions` CLI, or the console — they belong
to the project, not to a signed-in user.

## Background work model

Actions are **the** way to run background and scheduled work on Forte:

1. **No CPU between requests.** Services keep memory provisioned (no cold starts), but CPU is scaled to zero
   whenever no request is in flight. In-process schedulers (`@Scheduled`, `ScheduledExecutorService`,
   `setInterval`, node-cron, APScheduler, Celery beat), polling loops, worker processes, and work continued after
   the response (fire-and-forget promises or threads, in-memory queues) won't run reliably. Replace them with Actions.
2. **Actions call a route on your service.** They fire on a recurring cron schedule, or once at a future time,
   and the service autoscales to handle invocations. From code, you can create a one-time action for
   per-entity deferred work (e.g. a follow-up 24h after sign-up).
3. **Do the work inside the request, then respond.** Each invocation gets **120 seconds**, and the `2xx` should
   mean "done". That's more than people expect: a batch of emails, a page of an API sync, a report, a
   cleanup sweep, LLM calls.
4. **Chunk bigger jobs.** Keep progress in the database, not in memory:
   - **Recurring sweeper**: an every-minute action processes the next N rows with a pending status or due time.
     Best for continuous work, and for high-volume per-entity scheduling.
   - **Self-chaining**: process one chunk, save a cursor, then call `createActionInvocation` on the same action
     for the next chunk, and stop when there's nothing left. Invoke-now reuses the configured body, so the
     cursor must live in your DB, and the action must be enabled.
   - **One-time action per task**: fine at low volume. Unrun one-time actions count toward the enabled-action
     quota (a fired one-time action is marked completed and stops counting), so switch to a sweeper at scale.
5. **Be idempotent.** Dedupe on `X-Forte-Action-Invocation-Id`, and make each chunk safe to re-run.
6. **Not supported yet: long-running, persistent, or stateful workers.** That covers queue consumers, always-on
   bots and socket listeners, and jobs that can't be split into pieces under 120s. Don't invent a workaround.
   Tell the customer Forte is building queue-based, longer-running background workers, and to contact
   support@forteplatforms.com for early access or to talk through the workload.

## Schedule types

- **Recurring** — a cron expression evaluated in an IANA timezone (e.g. `America/New_York`), with an
  optional active window (start/end). An action fires **at most once per minute**; sub-minute schedules
  are rejected (`ACTION_SCHEDULE_TOO_FREQUENT`).
- **One-time** — a single future timestamp (up to about a year out). The action fires once, then is marked
  completed (`enabled: false`) and no longer counts toward the quota.

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
- `X-Request-Id` — matches the invocation's entry in the service's request logs.

Forte delivers the request over its private network, bypassing user auth, so **don't add an auth exclusion**
for `targetPath` — the `X-Forte-Trusted` check is what protects it. Make handlers **idempotent on the invocation id**. Do the work **before** returning `2xx`,
within the **120-second** limit. Past that, the attempt is recorded as a timeout.

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

Handler on the target service (verify the trusted header, dedupe on the invocation id, do the work, then return 2xx):

```typescript
app.post("/jobs/nightly", async (req, res) => {
  if (req.header("X-Forte-Trusted") !== "1") return res.sendStatus(401);
  const invocationId = req.header("X-Forte-Action-Invocation-Id");
  if (await alreadyProcessed(invocationId)) return res.sendStatus(200);
  await runNightlyReport(req.body); // inside the request: no CPU after the response
  await markProcessed(invocationId);
  res.sendStatus(200);
});
```

Self-chaining handler for a job that won't fit in 120s:

```typescript
app.post("/jobs/reindex", async (req, res) => {
  if (req.header("X-Forte-Trusted") !== "1") return res.sendStatus(401);
  const job = await db.jobs.findOne({ name: "reindex" });
  const batch = await db.documents.find({ _id: { $gt: job.cursor } }).sort({ _id: 1 }).limit(500);
  for (const doc of batch) await reindex(doc);
  if (batch.length > 0) {
    await db.jobs.updateOne({ name: "reindex" }, { $set: { cursor: batch.at(-1)._id } });
    await forte.projects.createActionInvocation({
      projectId: process.env.FORTE_PROJECT_ID,
      actionId: req.header("X-Forte-Action-Id"),
    });
  }
  res.sendStatus(200);
});
```

Canonical docs: [forteplatforms.com/docs/core-concepts/actions](https://forteplatforms.com/docs/core-concepts/actions)
and [forteplatforms.com/docs/guides/creating-actions](https://forteplatforms.com/docs/guides/creating-actions)
