# Eve Channel Adapters

*Load this file when configuring channels — Slack, Discord, Teams, Telegram, GitHub,
Linear, web chat, or custom channels.*

---

## Channel Concepts

- HTTP is **always on** by default. No channel file required.
- Every other channel is one file in `agent/channels/`.
- Scaffold with `eve channels add <name>` or create the file manually.
- The same agent codebase serves all channels — one deploy, every surface.
- Sessions **move across channels**: a webhook on HTTP can continue as a Slack thread.

---

## Built-in Channel Adapters

### Slack

```typescript
// agent/channels/slack.ts
import { defineSlackChannel } from 'eve/channels/slack';

export default defineSlackChannel({
  // Credentials via Vercel Connect — no SLACK_BOT_TOKEN in .env required
  // when using Connect. For local dev, set SLACK_BOT_TOKEN env var.
});
```

**Capabilities**:
- Approval gates render as interactive Slack buttons.
- Typing indicators while agent is processing.
- Select menus for structured choices.
- Rich block kit formatting.
- Mentions, threads, and DMs all route to the agent.

**Scaffold**: `eve channels add slack`

---

### Discord

```typescript
// agent/channels/discord.ts
import { defineDiscordChannel } from 'eve/channels/discord';

export default defineDiscordChannel({
  // Set DISCORD_BOT_TOKEN env var, or route via Vercel Connect
});
```

---

### Microsoft Teams

```typescript
// agent/channels/teams.ts
import { defineTeamsChannel } from 'eve/channels/teams';

export default defineTeamsChannel({
  // Uses Azure Bot Service credentials
  // Set TEAMS_APP_ID and TEAMS_APP_PASSWORD env vars
});
```

---

### Telegram

```typescript
// agent/channels/telegram.ts
import { defineTelegramChannel } from 'eve/channels/telegram';

export default defineTelegramChannel({
  // Set TELEGRAM_BOT_TOKEN env var
});
```

---

### GitHub

```typescript
// agent/channels/github.ts
import { defineGithubChannel } from 'eve/channels/github';

export default defineGithubChannel({
  // Responds to GitHub events: issues, PRs, comments, webhooks
  // Set GITHUB_APP_ID, GITHUB_PRIVATE_KEY, GITHUB_WEBHOOK_SECRET
  // or route via Vercel Connect
});
```

---

### Linear

```typescript
// agent/channels/linear.ts
import { defineLinearChannel } from 'eve/channels/linear';

export default defineLinearChannel({
  // Responds to Linear issue and comment events
  // Set LINEAR_WEBHOOK_SIGNING_SECRET
  // or route via Vercel Connect
});
```

---

### Web Chat (Next.js)

```typescript
// agent/channels/web.ts
import { defineWebChannel } from 'eve/channels/web';

export default defineWebChannel({
  // Embeds a web chat UI powered by Next.js
  // Add only with: npx eve@latest init <name> --channel-web-nextjs
});
```

**Important**: Only use `--channel-web-nextjs` at `init` time if the user explicitly
wants a web UI. It adds Next.js project scaffolding; not needed for API or messaging use cases.

---

### Custom Channels

```typescript
// agent/channels/my-custom-channel.ts
import { defineChannel } from 'eve/channels';

export default defineChannel({
  name: 'my-custom-channel',
  async receive(message, context) {
    // Parse incoming events from your system
    return { content: message.body.text, userId: message.body.userId };
  },
  async send(message, context) {
    // Send the agent's reply back to your system
    await myApi.postMessage(context.userId, message.content);
  },
});
```

---

## Channel Handoff Pattern

One channel can trigger activity on another. Common use case: an incident webhook
arrives over HTTP, the agent opens an investigation thread in Slack.

```typescript
// agent/channels/incident-webhook.ts
import { defineChannel } from 'eve/channels';
import slack from './slack.js';

export default defineChannel({
  name: 'incident-webhook',
  async receive(event, context) {
    // Receive a raw webhook
    return { content: `Incident alert: ${event.body.description}` };
  },
  async onComplete(result, context) {
    // Post the agent's findings to Slack after it finishes
    await context.send(slack, {
      content: result.content,
      target: { channelId: 'C0INCIDENTS' },
    });
  },
});
```

---

## Credential Routing Summary

| Channel | Local dev | Vercel (production) |
|---|---|---|
| Slack | `SLACK_BOT_TOKEN` env var | Vercel Connect (OAuth, auto-refresh) |
| Discord | `DISCORD_BOT_TOKEN` env var | Vercel Connect or env var |
| Teams | `TEAMS_APP_ID` + `TEAMS_APP_PASSWORD` | Vercel Connect or env var |
| Telegram | `TELEGRAM_BOT_TOKEN` env var | Vercel Connect or env var |
| GitHub | GitHub App credentials | Vercel Connect (recommended) |
| Linear | `LINEAR_WEBHOOK_SIGNING_SECRET` | Vercel Connect (recommended) |

Use **Vercel Connect** for production to avoid bot tokens in environment variables and to get
automatic token refresh without code changes.
