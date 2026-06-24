---
name: vercel-eve-framework
description: >
  Expert guidance for building AI agents with Vercel's Eve framework — a filesystem-first,
  TypeScript-native, open-source agent framework (public beta since June 17, 2026).
  Use this skill whenever a user wants to build, scaffold, configure, test, deploy, or debug
  an Eve agent. Trigger on any mention of "Eve agent", "vercel eve", "eve dev", "eve framework",
  "npx eve@latest", agent directories with agent.ts / instructions.md, or requests to add tools /
  skills / channels / connections / schedules / subagents / evals to an Eve project. Also trigger
  when scaffolding a new AI agent on Vercel, migrating from another agent framework to Eve,
  or wiring up Vercel Sandbox, Workflows, AI Gateway, or Connect inside an agent project.
  This skill covers the complete Eve surface: project structure, all file primitives, CLI,
  HTTP API, durable sessions, approval gates, multi-channel deployment, subagents, evals,
  observability, and production deployment via `vercel deploy`.
---

# Vercel Eve Framework Skill

> **Status**: Public beta (launched June 17, 2026). API and behaviour may change before GA.
> **License**: Apache-2.0 | **npm**: `eve` | **GitHub**: `github.com/vercel/eve`
> **Docs**: `vercel.com/docs/eve` | once eve is installed, docs are also bundled at `node_modules/eve/docs`

---

## 1 · What Eve Is

Eve is a **filesystem-first framework for durable backend AI agents**. Its core insight:
*an agent has a shape — a directory of files — and that shape should be the framework.*

| Concept | Eve answer |
|---|---|
| Agent definition | A directory of TypeScript + Markdown files under `agent/` |
| Execution runtime | Compiles to Vercel Functions; durable via Vercel Workflows |
| State persistence | Checkpointed at every step; survives crashes, cold starts, deploys |
| Code execution | Isolated sandboxes (Vercel Sandbox in prod, Docker/microsandbox/just-bash locally) |
| Model routing | Vercel AI Gateway (any provider; automatic fallback; OIDC auth on Vercel) |
| External auth | Vercel Connect (OAuth, API keys, MCP; credentials never exposed to the model) |
| Observability | OpenTelemetry; Agent Runs tab in Vercel dashboard; exportable to Braintrust/Honeycomb/Datadog |
| Deployment | `vercel deploy` — exactly the same as any Vercel web project |

**"Next.js for agents"** — Eve owns the execution loop the way Next.js owns routing.
A file's name and location in the directory tree *is* its registration. No boilerplate.

---

## 2 · Project Structure (Complete Reference)

```
agent/
  agent.ts                   # required — model config and runtime options
  instructions.md            # required — system prompt prepended to every model call

  tools/
    <tool_name>.ts           # one file = one tool; filename becomes tool name
    ...

  skills/
    <skill_name>.md          # loaded dynamically when relevant topic arises
    ...

  subagents/
    <child_name>/            # nested agent directory; same shape as root agent/
      agent.ts
      instructions.md
      tools/
      ...

  channels/
    slack.ts                 # one file per channel; HTTP is on by default
    discord.ts
    teams.ts
    telegram.ts
    github.ts
    linear.ts
    web.ts                   # web chat (Next.js); use --channel-web-nextjs at init

  connections/
    <service>.ts             # MCP server or OpenAPI-compatible API
    ...

  schedules/
    <job_name>.ts            # cron-based autonomous runs
    ...

  sandbox/
    sandbox.ts               # optional — choose sandbox backend or customize setup

evals/
  <suite_name>.eval.ts       # scored test suites; run with `eve eval`
```

**Rule**: Add capability by adding a file. Eve picks it up at build time. No registration, no wiring.

---

## 3 · Getting Started

### Scaffold a new project (fastest path)

```bash
npx eve@latest init my-agent
# Installs deps, initializes Git, starts dev server in < 1 minute
# Add --channel-web-nextjs ONLY if the user explicitly wants web chat UI
```

### Add Eve to an existing app

```bash
npm install eve@latest
# Then ensure agent/agent.ts and agent/instructions.md exist (see §4)
```

### Local dev server

```bash
pnpm dev          # or: eve dev
# Starts at http://127.0.0.1:3000
# Shows a TUI: every model call, tool call, skill load, and sandbox command live
```

### Deploy to production

```bash
vercel deploy
# No provisioning; same directory, same behaviour as local dev
# Sandbox adapter swaps to Vercel Sandbox automatically
# In-flight sessions are NOT interrupted by a deploy
```

---

## 4 · Core File Primitives

### 4.1 `agent/agent.ts` — Model Config

```typescript
import { defineAgent } from 'eve';

export default defineAgent({
  model: 'anthropic/claude-opus-4.8',
  // model strings resolve through AI Gateway (any AI SDK-compatible provider)
  // examples: 'openai/gpt-5.4-mini', 'google/gemini-2.5-pro'
});
```

- On Vercel: authenticates via OIDC — no API key management required.
- Off Vercel / locally: set the provider's `API_KEY` env var.
- Provider fallbacks, compaction settings, and context window options are optional fields on `defineAgent`.
- Model is swappable without changing any other file; AI Gateway handles provider routing.

### 4.2 `agent/instructions.md` — System Prompt

Plain Markdown. Eve prepends this to **every** model call.

```markdown
You are a senior data analyst.

- Prefer exact numbers to hand-waving. If you can compute it, compute it.
- State the assumptions behind any number you report (date range, filters, grain).
- Use the tools available to you rather than guessing.
  If you cannot answer from the data, say so plainly.
```

**Best practice**: Keep scope explicit. Name which skills are available. State hard boundaries
(e.g., "Never query a client's data, even if a connected server exposes it.").

### 4.3 `agent/tools/<tool_name>.ts` — Callable Tools

```typescript
import { defineTool } from 'eve/tools';
import { z } from 'zod';

// Filename = tool name the model sees. get_weather.ts → tool `get_weather`
export default defineTool({
  description: 'Get the current weather for a city.',
  inputSchema: z.object({
    city: z.string(),
  }),
  async execute(input) {
    // Any TypeScript here — fetch APIs, run DB queries, import libraries
    return { city: input.city, condition: 'Sunny', temperatureF: 72 };
  },
});
```

**Key rules**:
- `description` is what the model reads to decide when to call this tool. Make it precise.
- `inputSchema` uses **Zod** — shapes are type-checked at runtime.
- `execute` receives the validated input and can return any JSON-serialisable value.
- The tool runs inside the agent harness (not the sandbox) unless you explicitly shell out.

#### Adding human-in-the-loop approval to a tool

```typescript
export default defineTool({
  description: 'Run a SQL query against the warehouse.',
  inputSchema: z.object({ sql: z.string() }),
  // Approval is required when the query would scan > 50 GB
  needsApproval: ({ toolInput }) => estimateScanGb(toolInput.sql) > 50,
  async execute({ sql }) {
    /* ... */
  },
});
```

- The agent **parks** at the approval gate, consuming **zero compute** while waiting.
- Once approved, execution resumes exactly where it left off.
- On Slack, approvals render as interactive buttons automatically.
- Can also use `needsApproval: true` to always require approval.

### 4.4 `agent/skills/<skill_name>.md` — On-Demand Playbooks

```markdown
---
description: How this team defines revenue. Load before answering any revenue question.
---

Revenue is recognized net of refunds, over the subscription term.
Weeks are Monday-anchored, in UTC.
Exclude trial and internal accounts from every number.
```

- The `description` frontmatter is what Eve uses to decide when to load this skill.
- Skills are **not** always in context — only loaded when the topic is relevant.
- This keeps prompt tokens low for conversations that don't need this knowledge.
- Think of skills as focused playbooks: policy docs, domain definitions, step-by-step procedures.

### 4.5 `agent/connections/<service>.ts` — External Services

#### MCP server connection

```typescript
import { defineMcpClientConnection } from 'eve/connections';

export default defineMcpClientConnection({
  url: 'https://mcp.linear.app/sse',
  description: 'Linear workspace: issues, projects, cycles, and comments.',
  auth: {
    getToken: async () => ({ token: process.env.LINEAR_API_TOKEN! }),
  },
});
```

#### OpenAPI / REST connection

```typescript
import { defineApiConnection } from 'eve/connections';

export default defineApiConnection({
  openApiUrl: 'https://api.example.com/openapi.json',
  description: 'Internal orders API.',
  auth: {
    type: 'bearer',
    getToken: async () => process.env.ORDERS_API_KEY!,
  },
});
```

**Security**: Eve discovers remote tools, hands them to the model, and brokers auth.
The **model never sees the connection's URL or credentials**.
Use **Vercel Connect** for OAuth: consent flows and token refresh are built in.

**Supported at launch**: Slack, GitHub, Snowflake, Salesforce, Notion, Linear
(plus anything reachable over OAuth, API key, or any MCP server).

### 4.6 `agent/channels/<channel>.ts` — Deployment Surfaces

HTTP is always on. Add surfaces with channel files. One file = one channel.

```bash
eve channels add slack      # scaffolds channels/slack.ts
eve channels add discord
eve channels add teams
```

Scaffolded Slack channel file:

```typescript
import { defineSlackChannel } from 'eve/channels/slack';

export default defineSlackChannel({
  // Route credentials through Vercel Connect — no bot token in .env
});
```

**Channel capabilities**:
- Approval gates render as **interactive Slack buttons** automatically.
- Typing indicators, select menus, and rich formatting come with each adapter.
- Sessions **move between channels**: a webhook arriving over HTTP can open a Slack thread.
- `defineChannel` covers custom channels not in the built-in list.

### 4.7 `agent/schedules/<job_name>.ts` — Autonomous Cron Runs

```typescript
import { defineSchedule } from 'eve/schedules';
import slack from '../channels/slack.js';

export default defineSchedule({
  cron: '0 9 * * 1',   // every Monday at 09:00 UTC
  async run({ receive, waitUntil, appAuth }) {
    waitUntil(
      receive(slack, {
        message: 'Summarize last week\'s revenue and post it to the team channel.',
        target: { channelId: 'C0123ABC' },
        auth: appAuth,
      }),
    );
  },
});
```

On Vercel, each schedule deploys as a **Vercel Cron Job**. No additional setup required.

### 4.8 `agent/subagents/<name>/` — Delegation

A subagent is a **full nested agent directory** inside `subagents/`:

```
agent/subagents/investigator/
  agent.ts           # can use a different model from the parent
  instructions.md    # its own identity and rules
  tools/             # its own private tools
```

```typescript
// agent/subagents/investigator/agent.ts
import { defineAgent } from 'eve';

export default defineAgent({
  description: 'Investigates data anomalies before the analyst reports them.',
  model: 'anthropic/claude-opus-4.8',
});
```

- The parent calls the subagent like any other tool.
- The child starts with a **clean context window** and only its own tools.
- Result is returned to the parent on completion.
- Use subagents for specialized work that benefits from focused context and different model choices.

### 4.9 `agent/sandbox/sandbox.ts` — Compute Backend

Eve gives every agent a real shell. The sandbox backend is an adapter:

| Environment | Default backend |
|---|---|
| `vercel deploy` | Vercel Sandbox (isolated VMs) |
| `eve dev` (local) | Docker → microsandbox → just-bash (first available) |

Override the backend:

```typescript
import { defineSandbox } from 'eve/sandbox';

export default defineSandbox({
  backend: 'docker',   // or 'microsandbox', 'just-bash', or a custom adapter
});
```

The agent can run bash, write files, execute Python scripts, grep codebases — all fully isolated
from the harness that controls the agent. Treat all agent-generated code as untrusted.

---

## 5 · HTTP API (Sessions)

Eve exposes a structured HTTP API on `http://127.0.0.1:3000` locally
(and at your Vercel deployment URL in production).

### Create a session (start a conversation)

```bash
curl -X POST http://127.0.0.1:3000/eve/v1/session \
  -H 'content-type: application/json' \
  -d '{"message": "What was revenue last week?"}'
```

Response includes:
- `continuationToken` — use this to send follow-up messages
- `x-eve-session-id` header — use this to attach to the stream

### Attach to the live stream

```bash
curl http://127.0.0.1:3000/eve/v1/session/<sessionId>/stream
# Returns NDJSON lifecycle events: model calls, tool calls, skill loads, sandbox commands
```

### Send a follow-up message

```bash
curl -X POST http://127.0.0.1:3000/eve/v1/session/<sessionId>/message \
  -H 'content-type: application/json' \
  -H 'x-eve-continuation-token: <continuationToken>' \
  -d '{"message": "Break that down by region."}'
```

### Key session properties

- Sessions are **durable workflows**: each step is checkpointed.
- A session can pause indefinitely (approval gate, human response) with **zero compute cost**.
- Sessions survive crashes, cold starts, and new deployments.
- A deploy **does not interrupt** in-flight sessions; they finish on the version they started on.

---

## 6 · Evals (Testing)

```typescript
// evals/revenue.eval.ts
import { defineEval } from 'eve/evals';
import { includes } from 'eve/evals/expect';

export default defineEval({
  description: 'Analyst answers revenue questions by the team\'s rules.',
  async test(t) {
    await t.send('What was revenue last week?');
    t.completed();                            // assert the agent finished the task
    t.calledTool('run_sql');                  // assert the right tool was used
    t.check(t.reply, includes('net of refunds')); // assert output content
  },
});
```

```bash
eve eval                   # run against local dev server
eve eval --url <prod-url>  # run against a deployed agent
```

**Wire into CI**: Each eval suite becomes a deploy gate. A prompt change or model swap shows
breakage before users see it. Every commit also gets its own **preview deployment** with channels
attached, so teams can test the next version of the Slack bot before it goes live.

---

## 7 · Observability

### OpenTelemetry trace structure (per turn)

```
ai.eve.turn                      # one span per turn
├── ai.streamText                # the model call
│   └── ai.streamText.doStream
└── ai.toolCall                  # tool name, inputs, outputs
    └── sandbox.bash             # if the tool shells out
```

- Spans are standard OpenTelemetry — export to Braintrust, Honeycomb, Datadog, Jaeger, etc.
- On Vercel: spans surface in the **Agent Runs tab** under Observability (zero configuration).
- Each run shows sessions, turns, tools used, reasoning, timing, and token usage.
- **Replay**: reproduce the exact model calls, tool invocations, and sandbox commands a run made.

---

## 8 · Common Patterns

### Pattern A — Minimal working agent (two files)

```
agent/
  agent.ts          # model: 'openai/gpt-5.4-mini'
  instructions.md   # "You are a concise assistant. Use tools when available."
```

### Pattern B — Data analyst with SQL tool, revenue skill, Slack channel, weekly report

```
agent/
  agent.ts          # claude-opus-4.8 for judgment quality
  instructions.md   # analyst identity, data scope rules, hard boundaries
  tools/
    run_sql.ts      # read-only SELECT only; needsApproval if scan > 50 GB
    post_chart.ts   # posts PNG charts to Slack
  skills/
    revenue-definitions.md  # loaded only on revenue questions
  channels/
    slack.ts        # team-facing surface
  schedules/
    monday-summary.ts  # cron 0 9 * * 1 → posts to #revenue channel
```

### Pattern C — Multi-agent with investigator subagent

```
agent/
  agent.ts
  instructions.md
  subagents/
    investigator/   # separate model, separate context, separate tools
      agent.ts
      instructions.md
      tools/
        deep_search.ts
```

### Pattern D — External service connection (GitHub MCP)

```typescript
// agent/connections/github.ts
import { defineMcpClientConnection } from 'eve/connections';

export default defineMcpClientConnection({
  url: 'https://api.githubcopilot.com/mcp/',
  description: 'GitHub: repos, issues, PRs, code search.',
  auth: {
    getToken: async () => ({ token: process.env.GITHUB_TOKEN! }),
  },
});
```

---

## 9 · Key Gotchas & Guard Rails

1. **Eve is in public beta** — APIs and file conventions may change before GA.
   Always pin `eve@latest` in package.json during development; audit changelog before upgrades.

2. **`npx eve@latest init` starts a dev server** — run it in a controllable process and stop it
   before editing files. Re-start after edits to pick up changes.

3. **Tool filename = tool name** — use `snake_case` filenames. `runSql.ts` → tool `runSql`,
   not `run_sql`. Be intentional.

4. **`needsApproval` can be a function or `true`** — use the function form for conditional
   approvals (e.g., based on input size, destructive operation type).

5. **Skills have a `description` frontmatter field** — this is what Eve uses for relevance
   matching. If it is missing or vague, the skill may never be loaded.

6. **Model strings go through AI Gateway** — on Vercel, no API keys needed. Locally, set the
   relevant env var (e.g., `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`).

7. **Web chat channel (`--channel-web-nextjs`)** — only add this flag at `init` time if the user
   explicitly wants a web UI. It adds Next.js scaffolding and is not needed for API/Slack/Discord
   use cases.

8. **Do not commit unless the user asks** — after scaffolding or editing, run typechecks and
   verify the HTTP API works before committing. Use `pnpm typecheck` or `tsc --noEmit`.

9. **Sandbox runs as isolated compute** — agent-written code executes in the sandbox, not the
   harness. You can read/write files and run arbitrary scripts there; none of it can reach the
   harness's environment variables or file system.

10. **Porting Eve to non-Vercel infra** — technically possible via adapter pattern, but each
    Eve production capability (Workflows, Sandbox, AI Gateway, Connect) needs a replacement.
    This is non-trivial. For platform-neutral needs, consider Mastra (runs anywhere, also TypeScript).

---

## 10 · CLI Reference

| Command | Description |
|---|---|
| `npx eve@latest init <name>` | Scaffold new agent project, install deps, init Git, start dev server |
| `npx eve@latest init <name> --channel-web-nextjs` | Same, plus web chat UI (Next.js) |
| `npm install eve@latest` | Add Eve to an existing project |
| `pnpm dev` / `eve dev` | Start local dev server with TUI |
| `eve eval` | Run eval suites against local dev server |
| `eve eval --url <url>` | Run eval suites against deployed agent |
| `eve channels add <channel>` | Scaffold a channel file (slack / discord / teams / github / linear / telegram) |
| `vercel deploy` | Deploy agent to production (same as any Vercel project) |

---

## 11 · Quick-Reference Imports

```typescript
// Core
import { defineAgent } from 'eve';

// Tools
import { defineTool } from 'eve/tools';
import { z } from 'zod';                    // always use Zod for inputSchema

// Connections
import { defineMcpClientConnection } from 'eve/connections';
import { defineApiConnection } from 'eve/connections';

// Channels
import { defineSlackChannel } from 'eve/channels/slack';
import { defineDiscordChannel } from 'eve/channels/discord';
import { defineTeamsChannel } from 'eve/channels/teams';
import { defineChannel } from 'eve/channels';  // custom channels

// Schedules
import { defineSchedule } from 'eve/schedules';

// Sandbox
import { defineSandbox } from 'eve/sandbox';

// Evals
import { defineEval } from 'eve/evals';
import { includes, matches, contains } from 'eve/evals/expect';
```

---

## 12 · Coding-Agent Bootstrap Prompt

When a coding agent (Claude Code, Cursor, etc.) needs to scaffold or work on an Eve project,
give it this prompt:

```
Set up an Eve agent for the user. Eve is a filesystem-first TypeScript framework for durable
agents, published as the npm package eve.

Read its docs: once eve is installed they are bundled in the package at node_modules/eve/docs;
before eve is installed, read the published Introduction and Getting Started pages at
vercel.com/docs/eve and vercel.com/blog/introducing-eve.

If the project has no Eve app, scaffold one with `npx eve@latest init <name>`;
add `--channel-web-nextjs` only when the user explicitly wants Web Chat.
The init command installs dependencies, initializes Git, and starts the dev server,
so run it in a controllable process and stop it before editing.

To add Eve to an existing app, run `npm install eve@latest`.

Make sure agent/agent.ts and agent/instructions.md exist, then add a first typed tool at
agent/tools/get_weather.ts using defineTool from eve/tools with a Zod inputSchema and an
inline execute function.

Start the dev server, then exercise the HTTP API:
1. Create a session with POST /eve/v1/session
2. Attach to GET /eve/v1/session/:id/stream
3. Send a follow-up with the returned continuationToken

Verify with the project's typecheck, adapt model and provider choices to the project,
and do not commit unless the user asks.
```

---

## 13 · References

See `references/` directory for deeper dives:
- `references/vercel-infrastructure.md` — Workflows, Sandbox, AI Gateway, Connect, Observability
- `references/channel-adapters.md` — Per-channel configuration options and credential routing
- `references/eval-patterns.md` — Writing effective evals; CI integration patterns

**Official sources** (always authoritative over this skill):
- Docs: `vercel.com/docs/eve`
- Blog: `vercel.com/blog/introducing-eve`
- GitHub: `github.com/vercel/eve`
- Bundled docs (post-install): `node_modules/eve/docs`
