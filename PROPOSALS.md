# Proposed Solutions (Draft)

> **Status: proposals for review. None of these are approved decisions.**
>
> This document answers the open questions in [`ARCHITECTURE.md`](ARCHITECTURE.md). Each section gives a recommendation, the reasoning behind it, pros and cons, and the signal that would change the answer. It is not an implementation plan.

## At a glance

| Question | Current proposal | Confidence |
|---|---|---|
| Runner shape | One Cloud Run Job per turn | High for low traffic |
| Workspace | Snapshot plus clone fallback | Medium |
| Follow-ups | Firestore inbox, polled by the active job | High for v1 |
| GitHub auth | GitHub App in production; PAT for the prototype | High |
| Multi-workspace | Defer, but namespace by `team_id` now | High |
| Progress UX | Status updates plus a final answer; no token streaming in v1 | Medium |
| Sandboxing | Cloud Run isolation plus least privilege and egress limits | Medium |
| Retention and cost | 30-day state, 45-minute runs, token and cost caps | Medium |

## 1. Runner shape: Cloud Run Job or service worker?

**Proposal:** one Cloud Run Job execution per turn. The gateway starts the job and never holds the agent request.

**Why:** a job is a better fit for a long, isolated, one-shot process than a service request. It has no 60-minute request ceiling, no CPU-throttling-after-response problem, and no Pub/Sub push acknowledgement deadline to work around. It also gives each run its own container, which is useful when the agent executes code.

| Pros | Cons |
|---|---|
| No request lifetime limit; jobs support long task timeouts | Cold start: image pull plus agent startup can add several seconds |
| One execution maps to one run, so isolation and cleanup are simple | No inbound request channel; follow-ups need a state-based mailbox |
| Per-second billing only while running | Job execution concurrency and quotas need monitoring |
| No ack-deadline or duplicate-delivery semantics to design around | More moving parts than a single service (gateway, job, reconciler) |
| Crash leaves a clear failed execution to inspect | Debugging spans two deployables and Firestore |

**Evidence:** Cloud Run services cap requests at 60 minutes; Cloud Run Jobs allow much longer task timeouts (up to 24 hours). Pub/Sub push subscriptions have a maximum acknowledgement deadline of 10 minutes, and Cloud Tasks has a 30-minute dispatch deadline. Those limits are exactly what makes "ack, then finish the work later" awkward in a service worker.

**What would change this:** a hard requirement for sub-second follow-up steering, or a need to receive external webhooks mid-run. In that case, a small always-on control service or a per-thread actor becomes worth the cost. For low-traffic Slack use, it is not.

## 2. Workspace strategy: snapshot, clone, or mount?

**Proposal:** start with a workspace snapshot in GCS, plus a shallow clone fallback.

- Restore `workspaces/{thread}/latest.tgz` if it exists.
- Otherwise shallow-clone the configured repo.
- Snapshot source and `.git`, but exclude `node_modules`, caches, and build output.
- Skip the snapshot above a size cap and fall back to a clone on the next run.

**Why:** a coding agent often leaves uncommitted changes, so clone-only loses work between turns. A full snapshot preserves that work without introducing shared infrastructure. Excluding dependency directories keeps the common case small.

| Pros | Cons |
|---|---|
| Preserves uncommitted files and git state | Tar and upload cost grows with workspace size |
| No always-on disk, no new infrastructure | Large repos can exceed `/tmp` or the snapshot cap |
| Simple to reason about: one object per thread | Restoring a huge tree can dominate cold-start time |
| Works with the runner's ephemeral disk model | Snapshot timing and consistency need care on abort |

**Alternatives:**

- **Clone every run.** Cleanest and cheapest for committed work, but loses uncommitted changes. Good fallback when snapshots are too large.
- **GCS FUSE mount.** Convenient for large trees, but commonly slower for metadata-heavy workloads like `node_modules` and git. Benchmark before adopting.
- **Filestore/NFS.** Fast shared disk, but reintroduces always-on infrastructure and cross-run isolation questions.

**Evidence:** Cloud Run's local disk is ephemeral and memory-backed. The first attempt avoided Cloud Run for v1 specifically because SQLite needed a local persistent disk and Socket Mode needed an always-on process. This proposal avoids that dependency by keeping the durable state in GCS instead of a local database.

**What would change this:** repositories regularly above a few gigabytes, or a need for a warm dependency cache across runs. Then a mounted volume or a separate build cache is worth revisiting.

## 3. Follow-up latency: how does a busy thread receive a new message?

**Proposal:** a Firestore inbox per thread, polled by the active job every 2–3 seconds. The job steers the agent if it is still working, or drains the inbox as a new prompt after the current turn. `finishTurn` checks the inbox in the same transaction before marking the thread idle.

**Why:** Cloud Run has no sticky routing, so the gateway cannot address the job that owns a thread. A shared mailbox is the simplest reliable answer. It also survives job restarts and works if the run is briefly on a different instance.

| Pros | Cons |
|---|---|
| No sticky sessions, no service discovery | Up to a few seconds of latency before the agent sees a follow-up |
| Survives job crashes and retries | Polling adds Firestore reads |
| Atomic finish prevents losing a message during finalization | A message arriving during a long tool call waits until the agent yields |
| Same mechanism handles cancel requests and queued messages | Not a true interrupt; it is cooperative steering |

**Alternatives:**

- **Pub/Sub pull subscription per run.** Lower latency and ordered delivery, but adds lease management and another moving piece.
- **Always-on per-thread actor.** Best interactivity, worst cost and operational fit for low traffic.
- **No steering at all.** Simpler, but users expect a Slack conversation to accept a correction while the agent works.

**Evidence:** the first attempt used an in-process `asyncio.Lock` per thread, which is correct for a single always-on process but does not work across serverless instances. Firestore transactions are the cross-process replacement: one active run per thread, an inbox for everyone else, and an atomic finish.

**What would change this:** user testing showing that a 2–3 second delay is unacceptable, or polling reads becoming a meaningful cost. Then move to a pull subscription.

## 4. GitHub authentication: PAT or GitHub App?

**Proposal:** GitHub App with short-lived installation tokens for production; a personal access token for the first single-workspace prototype.

**Why:** a PAT is the fastest way to prove the integration, but it is tied to a person and usually carries broader scope than the task needs. A GitHub App can be installed on selected repositories, mints installation tokens that expire after about an hour, and is auditable per installation.

| Option | Pros | Cons |
|---|---|---|
| PAT | Minutes to set up; works with any MCP server or git clone | Broad scope; tied to a person; rotation is manual; poor multi-tenant story |
| GitHub App | Scoped to selected repos; short-lived tokens; clean audit trail; revocable per installation | More setup: app ID, private key, installation ID, token minting, refresh for long runs |

**Evidence:** GitHub installation access tokens expire after one hour. That is shorter than a generous agent run budget, so the runner must be able to refresh a token or accept that very long runs fail closed.

**What would change this:** if the bot is strictly single-user and the PAT is scoped to a read-only account, a PAT is defensible for a prototype. As soon as multiple workspaces or write access are involved, the App is the right default.

## 5. Multi-workspace installs: now or later?

**Proposal:** defer multi-workspace OAuth until single-workspace reliability is proven, but namespace all state by `team_id` from day one.

- Every thread key already includes `team_id`.
- Keep an installation record per team, even if the first version has one row.
- Put the token behind a resolver, not a global environment variable.
- Add per-team policy later: allowed repos, backend, budget, retention.

**Why:** OAuth installation storage, token rotation, per-tenant isolation, and billing add real surface area. None of it is needed to validate the agent experience in one workspace. Namespacing now is cheap and avoids a painful migration later.

| Pros of deferring | Cons of deferring |
|---|---|
| Smaller first version; focus on agent quality | Retrofitting token storage and policy touches every request path |
| No OAuth edge cases or marketplace requirements | Early adopters must be added manually |
| Easier security review with one tenant | Usage accounting is simpler but less general |

**Evidence:** the first attempt explicitly put multi-workspace OAuth out of scope for v1. That was a reasonable call. Its state was also single-workspace by design, which is the part this proposal avoids repeating.

**What would change this:** a plan to publish the app, or a second workspace asking to use it before the first is stable.

## 6. Progress UX: token streaming or status updates?

**Proposal:** no token streaming in v1. Use an immediate `:eyes:` reaction, one progress message updated at most every ~2 seconds, and a final answer that replaces it. Split long answers into thread replies. Show the current tool or phase, not raw model output. Revisit streaming the final assistant turn only after user feedback.

**Why:** a coding agent produces long silences and useful tool activity. Streaming tokens creates a lot of Slack API traffic and a renderer that must handle throttling, chunking, fence repair, failed updates, and "text never shrinks" invariants. The first attempt built that renderer well, but it was serving a chat completion. Here, "running tests" is more informative than a partial sentence.

| Pros | Cons |
|---|---|
| Low rate-limit risk | Feels less alive during a long final answer |
| Readable, stable progress instead of flickering text | A slow tool can look like a stuck bot without a status line |
| Much less renderer complexity in the runner | Less impressive in a demo |
| Final answer is clean and complete | Users cannot interrupt based on partial text |

**What to borrow from the first attempt:** the placeholder message, throttled `chat.update`, chunking with fence repair, and status markers are all good Slack-specific ideas. The first attempt's `render.py` is the most reusable reference in that repository.

**Evidence:** Slack's `chat.update` is a Tier 3 method (roughly one update per second per channel), and message text is capped around 4000 characters. Token streaming multiplies update attempts and failure modes; status updates do not.

**What would change this:** user testing showing that long final answers feel unresponsive. Then stream only the final assistant turn, with the same throttle and chunking rules.

## 7. Sandboxing: is Cloud Run enough?

**Proposal:** treat the Cloud Run container as the baseline isolation boundary, not as a complete security sandbox.

- Use a dedicated, least-privilege service account.
- Keep the job private; no inbound ingress.
- Scope model, GitHub, and Slack tokens to the run.
- Add egress restrictions if untrusted users or repositories are allowed.
- Log tool calls and network destinations.
- Revisit stronger isolation if the bot becomes multi-tenant or touches sensitive repositories.

**Why:** the main risks are not container escape. They are prompt injection, broad credentials, and unrestricted network egress. A container boundary helps, but a token that can push to every repository is still dangerous inside a perfect sandbox.

| Pros | Cons |
|---|---|
| Managed isolation with no host mounts | Not a guarantee against a determined container escape |
| Ephemeral disk limits persistence of an intrusion | Agent code can still exfiltrate data over the network |
| Narrow IAM is straightforward with a dedicated job identity | Per-run token scoping adds work to the runner and gateway |
| No inbound path to the job | Debugging egress restrictions takes effort |

**Evidence:** the first attempt already treated MCP results as untrusted external content and clipped them. That instinct should carry over, but the threat model is larger here because the agent can run arbitrary code and edit repositories.

**What would change this:** a public or multi-tenant deployment, sensitive source code, or a compliance requirement. Then add a network policy, short-lived credentials, and possibly a nested sandbox for tool execution.

## 8. Retention and cost: how long does state live?

**Proposal:**

| Item | Proposed default |
|---|---|
| Event dedupe records | 24–48 hours |
| Run records | 30 days |
| Threads and sessions | 30 days after last activity |
| Workspace snapshots | 7–30 days |
| Artifacts | 7–30 days |
| Wall-clock run budget | 45 minutes, hard cap below the job timeout |
| Model budget | Per-run token/cost cap, plus a per-thread daily cap |

**Why:** Firestore and GCS are cheap relative to model tokens. The expensive failure is a runaway agent, not a stored session. A wall-clock budget plus a token cap gives two independent brakes. Retention should be long enough that a thread feels continuous for a month, but not indefinite.

| Pros | Cons |
|---|---|
| Predictable cost ceiling per run | A hard budget can cut off a legitimately long task |
| Bounded storage growth | Users may expect a thread to live longer than 30 days |
| Simple policy to explain and implement | Per-thread and per-user budgets need accounting |
| Short dedupe TTL keeps Firestore small | Aggressive cleanup can remove useful context |

**Evidence:** the first attempt invested heavily in token budgeting and context compaction because unbounded context was both a cost and quality problem. The same discipline applies here: a full agent can burn far more tokens than a completion, so the budget must be explicit.

**What would change this:** compliance retention requirements, or users who treat threads as durable project workspaces. Then pin selected threads or move them to a longer retention tier.

## What to borrow from the first attempt (Vaultnel)

The first attempt is **not** an architectural template. It is a single always-on Python process with Socket Mode, SQLite, and a direct LLM loop. Its `llm/agent.py` bounded tool loop and memory tiers are specific to that design and are not reusable for a full coding agent. The Slack wrapper, however, contains several patterns worth adapting. Treat it as a reference for Slack behavior, not code to copy.

| Pattern in Vaultnel | Keep? | Serverless adaptation |
|---|---|---|
| Ack-first handlers | Yes | The Bolt for JS equivalent is `ack()` followed by short asynchronous dispatch work. |
| Filter bots, subtypes, and own messages | Yes | Same rules, applied before admission. |
| Dedupe by `event_id` | Yes, but not in memory | Use Firestore so dedupe works across gateway instances and restarts. |
| Per-thread `asyncio.Lock` | No | Use one active run per thread in Firestore; in-process locks do not span instances. |
| `:eyes:` ack, then ✅ or ⚠️ | Yes | Cheap, reliable, and independent of the progress message. |
| Placeholder plus throttled `chat.update` | Yes | The runner owns it and feeds it normalized agent events. |
| Chunking with code-fence repair | Yes | Slack-specific and still necessary; reuse the idea, not the code. |
| Trailing status markers | Maybe | Useful fallback; reactions or Block Kit may be cleaner. |
| Display-name resolution | Yes, low priority | Cache per run; it is decoration, not correctness. |
| Reaction-based reset with a message-to-thread map | Yes | Store the mapping in Firestore so it survives the job. |
| Slash commands and ephemeral replies | Yes | Keep `/agent` style commands; define status, cancel, reset. |
| Socket Mode | No | Use the HTTP Events API; a Cloud Run service should not depend on a long-lived socket. |
| SQLite and local memory tiers | No | Pi's session JSONL plus GCS replaces the transcript store. |
| Direct LLM loop and tool agent | No | The runner drives a full agent backend over RPC. |
| Token-by-token streaming | Not in v1 | Status updates plus a final answer; reconsider after user feedback. |

The most useful borrowing is the **Slack protocol discipline**: acknowledge immediately, filter aggressively, serialize a thread, keep updates cheap, and make failure non-fatal. The least useful borrowing is the **runtime model**: one process, one socket, one local database.

## What this document does not decide

These remain open until the shape above is agreed:

- Firestore collection names and exact fields
- Repository and package layout
- Whether gateway and runner share an image
- Deployment tooling
- Default model and provider
- Whether the agent gets a Slack tool extension or only a final-answer channel

## References and evidence

Platform limits and behaviors:

- Slack Events API retries and the 3-second acknowledgement window: <https://api.slack.com/apis/events-api#retries>
- Slack `chat.update` rate limits: <https://api.slack.com/methods/chat.update>
- Slack message text limits: <https://api.slack.com/reference/messaging/payload>
- Cloud Run request timeout: <https://cloud.google.com/run/docs/configuring/request-timeout>
- Cloud Run Job task timeout: <https://cloud.google.com/run/docs/configuring/jobs/timeouts>
- Pub/Sub push acknowledgement deadline: <https://cloud.google.com/pubsub/docs/subscriber#push-subscription-properties>
- Cloud Tasks dispatch deadline: <https://cloud.google.com/tasks/docs/reference/rest/v2/projects.locations.queues.tasks>
- GitHub App installation tokens: <https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app>

First-attempt evidence (`~/projects/slackbot-vaultnel`):

- `src/slackbot_vaultnel/slack/app.py` — ack-first lazy listeners, handler wiring
- `src/slackbot_vaultnel/slack/handlers.py` — filtering, dedupe, thread locks, reactions, reset mapping
- `src/slackbot_vaultnel/slack/render.py` — placeholder, throttled updates, chunking, fence repair
- `docs/decisions.md` — Socket Mode and single-process tradeoffs, streaming rationale
- `docs/spec.md` — Slack behavior and message limits

Platform limits should be re-checked before implementation; they are cited as of this writing.
