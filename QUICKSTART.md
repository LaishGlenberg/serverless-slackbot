# Quickstart: MVP

> **This is the shortest path to a working Slack coding agent, not the final architecture.**
>
> It uses one Cloud Run service and one always-warm instance. It deliberately skips Firestore, Cloud Run Jobs, steering, and workspace snapshots. Build this first, prove the Slack + agent loop, then upgrade along the path at the end.

## What you get

- `@mention` the bot in a channel, or DM it.
- It acknowledges immediately and runs a real pi coding agent in the same container.
- It streams a simple elapsed-time status and posts the final answer in the thread.
- The pi session is saved to GCS, so the conversation continues after a cold start.
- Optionally, it shallow-clones a public repo into an ephemeral workspace.

## What you give up

- One instance only. No horizontal scaling.
- One agent run at a time. Messages queue in memory.
- Queued messages wait silently behind the current run.
- In channels, every follow-up needs a new `@mention`; DMs are continuous.
- Workspace changes are lost if the container restarts.
- No mid-run steering, cancel, reaction reset, or multi-workspace OAuth.
- No Firestore, no Cloud Run Job, no reconciler.

That is intentional. The code shape is still Slack handler → agent → state, so the upgrade is a move, not a rewrite.

## Prerequisites

- Node 24 and npm
- A Google Cloud project with billing enabled
- `gcloud` authenticated
- A Slack workspace where you can install an app
- An API key for a model pi supports (Anthropic, OpenAI, etc.)

---

## 1. Create the Slack app

Create an app at <https://api.slack.com/apps> → **Create New App** → **From an app manifest**. Use this manifest, then install it to your workspace.

```yaml
display_information:
  name: slack-agent
  description: Coding agent in a Slack thread
  background_color: "#1f2937"

features:
  app_home:
    home_tab_enabled: false
    # Required for DMs; without it Slack refuses to send messages to the app.
    messages_tab_enabled: true
    messages_tab_read_only_enabled: false
  bot_user:
    display_name: slack-agent
    always_online: true

oauth_config:
  scopes:
    bot:
      - app_mentions:read   # @mentions in channels
      - chat:write          # post and update messages
      - im:history          # read DMs
      - im:read             # DM metadata
      - im:write            # reply in DMs

settings:
  event_subscriptions:
    bot_events:
      - app_mention
      - message.im
  interactivity:
    is_enabled: false
  socket_mode_enabled: false
```

After installing, copy:

- **Bot User OAuth Token** (`xoxb-...`) → `SLACK_BOT_TOKEN`
- **Signing Secret** from **Basic Information** → `SLACK_SIGNING_SECRET`

Do not set the Request URL yet. You will set it after the service is deployed.

---

## 2. Create the GCP resources

```bash
export PROJECT="your-project-id"
export REGION="us-central1"
export BUCKET="$PROJECT-slack-agent-mvp"

gcloud config set project "$PROJECT"

gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  storage.googleapis.com \
  secretmanager.googleapis.com

gcloud storage buckets create "gs://$BUCKET" --location="$REGION"
```

Create the secrets. Paste the values when prompted by the `read` commands.

```bash
read -r -s -p "Slack signing secret: " SLACK_SIGNING_SECRET; echo
read -r -s -p "Slack bot token: " SLACK_BOT_TOKEN; echo
read -r -s -p "Model API key: " MODEL_API_KEY; echo

printf '%s' "$SLACK_SIGNING_SECRET" | gcloud secrets create slack-signing-secret --data-file=-
printf '%s' "$SLACK_BOT_TOKEN"     | gcloud secrets create slack-bot-token --data-file=-
printf '%s' "$MODEL_API_KEY"       | gcloud secrets create model-api-key --data-file=-
```

If a secret already exists, add a new version instead:

```bash
gcloud secrets versions add slack-signing-secret --data-file=-
```

Create a dedicated runtime service account and give it only what the MVP needs.

```bash
gcloud iam service-accounts create slack-agent-mvp \
  --display-name="Slack agent MVP"

export SA="slack-agent-mvp@$PROJECT.iam.gserviceaccount.com"

for secret in slack-signing-secret slack-bot-token model-api-key; do
  gcloud secrets add-iam-policy-binding "$secret" \
    --member="serviceAccount:$SA" \
    --role="roles/secretmanager.secretAccessor"
done

gcloud storage buckets add-iam-policy-binding "gs://$BUCKET" \
  --member="serviceAccount:$SA" \
  --role="roles/storage.objectAdmin"

gcloud artifacts repositories create slack-agent \
  --repository-format=docker \
  --location="$REGION"
```

If the Artifact Registry repository already exists, skip that last command.

---

## 3. Add the MVP code

From the repository root:

```bash
npm install \
  @slack/bolt \
  @slack/web-api \
  @google-cloud/storage \
  @earendil-works/pi-coding-agent \
  tsx
```

Add the start script to `package.json`:

```json
{
  "scripts": {
    "start": "tsx src/index.ts"
  }
}
```

Create the three files below. The TypeScript entry point is in the appendix; it is intentionally a single file.

**`Dockerfile`**

```dockerfile
FROM node:24-bookworm-slim

RUN apt-get update \
  && apt-get install -y --no-install-recommends bash ca-certificates git ripgrep \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

ENV NODE_ENV=production
CMD ["npm", "start"]
```

**`.dockerignore`**

```text
node_modules
.git
.env
dist
*.log
```

**`src/index.ts`**

Copy the appendix into this file. It is a sketch, not production code.

---

## 4. Build and deploy

```bash
gcloud builds submit \
  --tag "$REGION-docker.pkg.dev/$PROJECT/slack-agent/mvp:latest" \
  .

gcloud run deploy slack-agent-mvp \
  --image "$REGION-docker.pkg.dev/$PROJECT/slack-agent/mvp:latest" \
  --region "$REGION" \
  --allow-unauthenticated \
  --service-account "$SA" \
  --min-instances 1 \
  --max-instances 1 \
  --no-cpu-throttling \
  --concurrency 10 \
  --cpu 2 \
  --memory 2Gi \
  --timeout 3600 \
  --set-env-vars "STATE_BUCKET=$BUCKET,AGENT_PROVIDER=anthropic,AGENT_MODEL=YOUR_MODEL_ID" \
  --set-secrets "SLACK_SIGNING_SECRET=slack-signing-secret:latest,SLACK_BOT_TOKEN=slack-bot-token:latest,ANTHROPIC_API_KEY=model-api-key:latest"
```

Replace `YOUR_MODEL_ID` with a model your key can use. If you are using OpenAI instead of Anthropic, change `AGENT_PROVIDER` and expose `OPENAI_API_KEY` instead of `ANTHROPIC_API_KEY`.

Optional: add `WORKSPACE_REPO_URL=https://github.com/you/public-repo` to `--set-env-vars` to shallow-clone a public repo into the workspace. Private repos need a credential helper and are deliberately out of scope for this MVP.

`--min-instances 1` and `--no-cpu-throttling` are not free: they keep CPU allocated so the work after the Slack ack survives. That is the price of this shortcut.

The flags that matter:

| Flag | Why |
|---|---|
| `--min-instances 1` | Keep the single instance warm; the agent runs after the Slack ack. |
| `--max-instances 1` | Keep the in-memory queue and `/tmp` workspace consistent. |
| `--no-cpu-throttling` | Allow work to continue after the HTTP response is sent. |
| `--concurrency 10` | Let Slack requests be acknowledged while an agent run is in progress. |

---

## 5. Wire Slack and test

Get the service URL:

```bash
gcloud run services describe slack-agent-mvp \
  --region "$REGION" \
  --format='value(status.url)'
```

In the Slack app:

1. Go to **Event Subscriptions**.
2. Set **Request URL** to `https://YOUR-SERVICE-URL/slack/events`.
3. Wait for **Verified**.
4. If you changed scopes, reinstall the app.

Test:

- DM the bot: `hello`.
- In a channel, invite the bot and mention it: `@slack-agent what files are in the workspace?`

Watch the logs:

```bash
gcloud run services logs read slack-agent-mvp --region "$REGION" --limit 50
```

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Slack Request URL will not verify | Signing secret, public ingress, `https://.../slack/events`, service logs. |
| Bot does not answer a mention | Bot is in the channel, `app_mention` subscription, scopes, app reinstalled. |
| Bot does not answer DMs | `message.im` subscription, `im:history`, app reinstalled. |
| `pi` fails to start | The pi package is installed, the `piCli` path is correct, container logs show the spawn error. |
| Session does not resume | `STATE_BUCKET`, service account object permissions, the GCS object name in logs. |
| Run starts but never finishes | Model key/provider/model, egress, Cloud Run CPU allocation, logs. |
| Answer arrives twice | Slack retry. Ack is sent first; add event dedupe if it persists. |
| Container restarts mid-run | The run is lost; the previous saved session survives. Check Cloud Run events. |
| `gcloud builds submit` is denied | Give the Cloud Build service account `roles/artifactregistry.writer` on the project or repository. |
| Model call fails immediately | The provider, model ID, and API key must match. Check the pi stderr in Cloud Run logs. |

---

## MVP limits and upgrade path

| MVP limit | Next step |
|---|---|
| One instance, one run at a time | Move the agent to a Cloud Run Job; gateway only acks and triggers. |
| Session persists, workspace does not | Snapshot the workspace tarball to GCS, or clone on every run. |
| No follow-ups while the agent works | Add a Firestore inbox and steer the active run. |
| No event dedupe | Store Slack `event_id`s in Firestore with a short TTL. |
| No cancel or reset | Add a Firestore flag and a reaction/command that sets it. |
| Single workspace | Add Slack OAuth installs and a per-team token resolver. |
| No Slack formatting conversion | Add a Markdown → Slack mrkdwn helper before posting. |
| Long answers split naively | Use fence-aware chunking so code blocks render correctly. |

---

## Appendix: minimum `src/index.ts`

> Sketch only. It has no dedupe, no workspace snapshot, no cancel, and a global run queue. It is meant to prove the loop.

```ts
import { App, ExpressReceiver } from "@slack/bolt";
import type { WebClient } from "@slack/web-api";
import { Storage } from "@google-cloud/storage";
import { RpcClient } from "@earendil-works/pi-coding-agent";
import { execFile } from "node:child_process";
import { existsSync } from "node:fs";
import { mkdir, readdir } from "node:fs/promises";
import { createRequire } from "node:module";
import { dirname, join } from "node:path";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);
const require = createRequire(import.meta.url);

const piPackage = require.resolve("@earendil-works/pi-coding-agent");
const piRoot = dirname(dirname(piPackage));
const piCli = [
  join(piRoot, "dist/bundle/cli.js"),
  join(piRoot, "dist/cli.js"),
].find((candidate) => existsSync(candidate));

if (!piCli) throw new Error("Could not find the pi CLI in the installed package");

const bucket = new Storage().bucket(process.env.STATE_BUCKET!);
const receiver = new ExpressReceiver({
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
  endpoints: "/slack/events",
});
const app = new App({
  token: process.env.SLACK_BOT_TOKEN!,
  receiver,
});

receiver.app.get("/healthz", (_req, res) => res.status(200).send("ok"));

// One run at a time. Good enough for an MVP, bad for a team.
let tail: Promise<unknown> = Promise.resolve();
function enqueue<T>(task: () => Promise<T>): Promise<T> {
  const run = tail.then(task, task);
  tail = run.then(
    () => undefined,
    () => undefined,
  );
  return run;
}

app.event("app_mention", async ({ ack, event, client }) => {
  await ack();
  const text = event.text.replace(/<@[^>]+>/g, "").trim();
  if (!text) return;

  const threadTs = event.thread_ts ?? event.ts;
  const key = `${event.channel}:${threadTs}`;
  void enqueue(() =>
    answer({ client, channel: event.channel, threadTs, text, key }),
  ).catch(console.error);
});

app.event("message", async ({ ack, event, client }) => {
  await ack();

  const message = event as {
    channel_type?: string;
    subtype?: string;
    bot_id?: string;
    channel: string;
    thread_ts?: string;
    text?: string;
  };

  if (message.channel_type !== "im" || message.subtype || message.bot_id) return;
  const text = message.text?.trim();
  if (!text) return;

  const key = `${message.channel}:dm`;
  void enqueue(() =>
    answer({
      client,
      channel: message.channel,
      threadTs: message.thread_ts,
      text,
      key,
    }),
  ).catch(console.error);
});

async function answer(args: {
  client: WebClient;
  channel: string;
  threadTs?: string;
  text: string;
  key: string;
}) {
  const { client, channel, threadTs, text, key } = args;
  const safe = key.replace(/[^a-zA-Z0-9._-]/g, "_");
  const workspace = `/tmp/workspaces/${safe}`;
  const sessionDir = `/tmp/pi-sessions/${safe}`;
  const sessionPath = join(sessionDir, "session.jsonl");
  const sessionObject = `sessions/${safe}.jsonl`;

  const placeholder = await client.chat.postMessage({
    channel,
    thread_ts: threadTs,
    text: "_Working…_",
  });

  const started = Date.now();
  const timer = setInterval(() => {
    const seconds = Math.round((Date.now() - started) / 1000);
    void client.chat
      .update({
        channel,
        ts: placeholder.ts!,
        text: `_Working… ${seconds}s_`,
      })
      .catch(() => {});
  }, 5000);

  let agent: RpcClient | undefined;

  try {
    await mkdir(workspace, { recursive: true });
    await mkdir(sessionDir, { recursive: true });

    if (
      process.env.WORKSPACE_REPO_URL &&
      (await readdir(workspace)).length === 0
    ) {
      await execFileAsync("git", [
        "clone",
        "--depth",
        "1",
        process.env.WORKSPACE_REPO_URL,
        workspace,
      ]);
    }

    const [hasSession] = await bucket.file(sessionObject).exists();
    if (hasSession) {
      await bucket.file(sessionObject).download({ destination: sessionPath });
    }

    const piArgs = [
      "--session-dir",
      sessionDir,
      "--append-system-prompt",
      "You are a coding agent replying in a Slack thread. Use Slack mrkdwn (not GitHub markdown). Be concise. The current directory is your workspace.",
    ];

    if (hasSession) {
      piArgs.push("--session", sessionPath);
    } else {
      piArgs.push("--session-id", safe);
    }

    agent = new RpcClient({
      cliPath: piCli,
      cwd: workspace,
      provider: process.env.AGENT_PROVIDER,
      model: process.env.AGENT_MODEL,
      args: piArgs,
    });

    await agent.start();
    await agent.promptAndWait(text);

    const final = (await agent.getLastAssistantText()) ?? "No answer.";
    const state = await agent.getState();

    if (state.sessionFile) {
      await bucket.upload(state.sessionFile, { destination: sessionObject });
    }

    const chunks = split(final);
    await client.chat.update({
      channel,
      ts: placeholder.ts!,
      text: chunks[0]!,
    });

    for (const chunk of chunks.slice(1)) {
      await client.chat.postMessage({
        channel,
        thread_ts: threadTs,
        text: chunk,
      });
    }
  } catch (error) {
    console.error(error);
    const message = error instanceof Error ? error.message : "Unknown error";
    await client.chat
      .update({
        channel,
        ts: placeholder.ts!,
        text: `:warning: ${message.slice(0, 3000)}`,
      })
      .catch(() => {});
  } finally {
    clearInterval(timer);
    if (agent) await agent.stop();
  }
}

function split(text: string, limit = 3500): string[] {
  const parts: string[] = [];
  let rest = text.trim();

  while (rest.length > limit) {
    let cut = rest.lastIndexOf("\n\n", limit);
    if (cut < limit / 2) cut = limit;
    parts.push(rest.slice(0, cut).trim());
    rest = rest.slice(cut).trim();
  }

  if (rest) parts.push(rest);
  return parts.length ? parts : ["(empty)"];
}

await app.start(Number(process.env.PORT ?? 8080));
console.log("slack agent listening");
```
