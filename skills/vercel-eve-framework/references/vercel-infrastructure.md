# Vercel Infrastructure Behind Eve

*Load this file when answering questions about Eve's production infrastructure,
Vercel Workflows, Sandbox, AI Gateway, Connect, or Observability configuration.*

---

## Vercel Workflows (Durable Execution)

Every Eve session is a **Vercel Workflow**. This is what makes sessions durable.

### What "durable" means in practice
- Every step in the agent loop is **checkpointed** as it completes.
- If the process crashes mid-turn, the workflow replays from the last checkpoint.
- Sessions can **park** between messages (e.g., waiting for a human approval)
  with zero compute consumed while parked.
- A `vercel deploy` does NOT interrupt running sessions; they finish on the version they started.

### Workflow SDK
Eve's durability is built on the open-source **Workflow SDK** (`workflow-sdk.dev`).
You generally do not interact with it directly — Eve wraps it — but it is available
if you need to write custom durable logic outside the agent loop.

### Cold start resilience
Sessions resume after cold starts automatically. There is no maximum pause duration.
An agent waiting for a human approval can wait days or weeks.

---

## Vercel Sandbox (Isolated Compute)

### Why it exists
Agent-generated code is untreated as **untrusted**. Eve runs it in a sandbox that is
isolated from the harness controlling the agent. The sandbox has its own file system,
environment, and network context.

### What agents can do in the sandbox
- Run bash commands
- Execute Python, Node, or any installed runtime
- Read and write files
- Install packages (within the sandbox)
- Grep codebases and run one-off analyses

### Adapter pattern
The sandbox backend is pluggable:

| Environment | Default adapter |
|---|---|
| `vercel deploy` | Vercel Sandbox (isolated VMs managed by Vercel) |
| `eve dev` local | Tries Docker → microsandbox → just-bash (first available) |

Override in `agent/sandbox/sandbox.ts` (see SKILL.md §4.9).

### Security boundary
- The sandbox **cannot** read the harness's environment variables.
- The sandbox **cannot** access the harness's file system.
- All communication between harness and sandbox is mediated by Eve.

---

## AI Gateway (Model Routing)

### What it does
- Single endpoint for multiple LLM providers (OpenAI, Anthropic, Google, etc.).
- Automatic **provider fallback**: if a model is unavailable, routes to the next.
- **OIDC authentication** on Vercel — no per-provider API keys to manage when deployed.
- Streams model responses with standard AI SDK conventions.

### Model string format
```
<provider>/<model-name>
```
Examples:
- `anthropic/claude-opus-4.8`
- `openai/gpt-5.4-mini`
- `google/gemini-2.5-pro`

Eve resolves these strings through AI Gateway at runtime.

### Local / off-Vercel auth
Set the relevant provider env var:
```
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GOOGLE_GENERATIVE_AI_API_KEY=...
```

### Model selection guidance
- **High capability (Opus-class)**: Use for judgment-heavy agents — data analysis, sales,
  content review. Weaknesses in the model translate to subtle wrong answers that guardrails cannot catch.
- **Fast/cheap (Mini-class)**: Use for routing agents, simple Q&A, tool-dispatch tasks.
- **Subagent model independence**: Each subagent can run a different model from its parent.
  A cheap routing agent can delegate to an expensive research subagent selectively.

---

## Vercel Connect (Credential Brokering)

### What it does
- Manages OAuth tokens and API keys for external services.
- Provides interactive OAuth consent flows and automatic **token refresh**.
- The **model never sees** the connection's URL or credentials — only the tools it exposes.

### Supported auth types (at launch)
- **OAuth (via Vercel Connect)**: Slack, GitHub, Salesforce, Notion, Linear, Snowflake
- **API key**: Any service (set in env or via Connect)
- **MCP**: Any MCP-compatible server

### Token lifecycle
Vercel Connect handles:
1. Initial OAuth consent (user-facing flow)
2. Token storage (encrypted, not in your code)
3. Token refresh (automatic, before expiry)

For API keys, use env vars locally and Vercel environment variables in production.
Never hardcode credentials in connection files.

---

## Vercel Observability (Agent Runs)

### Dashboard
- **Agent Runs tab** under Observability in the Vercel project dashboard.
- Shows: all sessions, turns, model calls, tool calls, reasoning, timing, token usage.
- **Replay**: reproduce the exact sequence of calls a past run made.
- No configuration required — all Eve agents get this automatically on Vercel.

### OpenTelemetry export
Eve emits standard OTel spans. Export to any compatible backend:
```
OTEL_EXPORTER_OTLP_ENDPOINT=https://api.honeycomb.io
OTEL_EXPORTER_OTLP_HEADERS="x-honeycomb-team=<your-key>"
```
Compatible backends: Braintrust, Honeycomb, Datadog, Jaeger, and any OTel-compatible collector.

### Span structure (per turn)
```
ai.eve.turn                    # wraps one full agent turn
├── ai.streamText              # the LLM call
│   └── ai.streamText.doStream
├── ai.toolCall                # one span per tool invocation
│   └── sandbox.bash          # if the tool ran shell commands
└── ai.skillLoad               # if a skill was loaded for this turn
```

Each span includes:
- **Inputs**: full prompt, tool input
- **Outputs**: model reply, tool return value
- **Timing**: start, end, duration
- **Token counts**: prompt tokens, completion tokens, total

### Token usage tracking
Token counts are visible per turn in the Agent Runs dashboard and in OTel spans.
Use this to optimize prompt length, skill loading frequency, and model selection.
