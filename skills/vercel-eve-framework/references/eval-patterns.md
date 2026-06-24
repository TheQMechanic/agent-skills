# Eve Eval Patterns

*Load this file when writing evals, setting up CI integration, or debugging
agent regressions with scored test suites.*

---

## Why Evals Matter for Agents

A change to `instructions.md` can break agent behaviour as surely as a code change.
Because an Eve agent lives in Git like software, it should be tested like software.

Evals let you:
- Verify the agent calls the right tools
- Assert that output follows business rules (e.g., "net of refunds")
- Catch regressions from model swaps or prompt edits before users see them
- Gate deploys in CI on scored checks

---

## Anatomy of an Eval

```typescript
// evals/revenue.eval.ts
import { defineEval } from 'eve/evals';
import { includes, matches, not } from 'eve/evals/expect';

export default defineEval({
  description: 'Analyst answers revenue questions by the team\'s rules.',

  async test(t) {
    // 1. Send a message to a fresh session
    await t.send('What was revenue last week?');

    // 2. Assert the session completed (did not error or time out)
    t.completed();

    // 3. Assert tool calls
    t.calledTool('run_sql');
    t.notCalledTool('post_chart');   // should not post a chart unprompted

    // 4. Assert output content
    t.check(t.reply, includes('net of refunds'));   // must follow revenue definition
    t.check(t.reply, not(includes('gross')));       // should never say "gross revenue"

    // 5. Multi-turn test
    await t.send('Break that down by region.');
    t.completed();
    t.calledTool('run_sql');   // must query again for regional breakdown
  },
});
```

---

## Available Assertions

### Session state
| Assertion | What it checks |
|---|---|
| `t.completed()` | Session finished without error or timeout |
| `t.failed()` | Session errored (use when testing error handling) |

### Tool calls
| Assertion | What it checks |
|---|---|
| `t.calledTool('tool_name')` | The named tool was called at least once |
| `t.notCalledTool('tool_name')` | The named tool was NOT called |
| `t.calledToolWith('tool_name', { key: value })` | Tool was called with specific input |
| `t.toolCallCount('tool_name', n)` | Tool was called exactly n times |

### Output content (use with `t.check`)
| Matcher | What it checks |
|---|---|
| `includes('text')` | Reply contains the string (case-insensitive) |
| `not(includes('text'))` | Reply does NOT contain the string |
| `matches(/regex/)` | Reply matches the regex |
| `contains(value)` | JSON reply contains the value (deep) |

---

## Eval Patterns

### Pattern 1: Business rule enforcement

```typescript
export default defineEval({
  description: 'Revenue definitions are always applied.',
  async test(t) {
    await t.send('What\'s our MRR?');
    t.completed();
    t.calledTool('run_sql');
    t.check(t.reply, includes('net of refunds'));
    t.check(t.reply, not(includes('trial')));   // trial accounts excluded
  },
});
```

### Pattern 2: Tool selection

```typescript
export default defineEval({
  description: 'Agent uses run_sql for data questions, not guesses.',
  async test(t) {
    await t.send('How many customers signed up yesterday?');
    t.completed();
    t.calledTool('run_sql');
    // Must not make up a number without querying
  },
});
```

### Pattern 3: Approval gate

```typescript
export default defineEval({
  description: 'Large SQL scans request approval.',
  async test(t) {
    await t.send('Run a full historical analysis of all orders ever.');
    t.pendingApproval('run_sql');   // should pause for approval on large queries
  },
});
```

### Pattern 4: Skill loading

```typescript
export default defineEval({
  description: 'Revenue skill is loaded for revenue questions.',
  async test(t) {
    await t.send('Define how we calculate ARR.');
    t.completed();
    t.loadedSkill('revenue-definitions');
  },
});
```

### Pattern 5: Multi-turn continuity

```typescript
export default defineEval({
  description: 'Agent maintains context across turns.',
  async test(t) {
    await t.send('What was revenue last week?');
    t.completed();
    const weeklyRevenue = t.reply;

    await t.send('Is that up or down from the week before?');
    t.completed();
    t.calledTool('run_sql');   // must re-query for comparison, not guess
    t.check(t.reply, includes('%'));   // should include a percentage change
  },
});
```

---

## Running Evals

```bash
# Run all eval suites against local dev server
eve eval

# Run against a deployed agent (e.g., a preview deployment)
eve eval --url https://my-agent-abc123.vercel.app

# Run a specific suite
eve eval evals/revenue.eval.ts

# Run with verbose output (shows each assertion result)
eve eval --verbose
```

---

## CI Integration (GitHub Actions Example)

```yaml
# .github/workflows/eval.yml
name: Agent Evals

on:
  push:
    branches: [main]
  pull_request:

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - run: pnpm install
      - run: pnpm dev &        # start dev server in background
      - run: sleep 5           # wait for server to be ready
      - run: eve eval          # run all evals
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

**Best practice**: Run evals on every PR. A failing eval blocks the merge.
Use `eve eval --url <preview-url>` to run evals against a Vercel preview deployment
(preview deployments are created automatically for every push).

---

## Debugging Regressions

When an eval fails after a change:
1. Check the Agent Runs tab in Vercel Observability for the failed session.
2. Use **Replay** to see the exact model calls, tool invocations, and skill loads.
3. Compare OTel spans before and after the change (timestamps in CI artifacts).
4. Roll back instructions.md or agent.ts and re-run evals to isolate the cause.
5. Use `eve eval --verbose` locally to see assertion-by-assertion results.

---

## Eval File Organization

For agents with many test suites, group by domain:

```
evals/
  data/
    revenue.eval.ts
    customer-count.eval.ts
    cohort-analysis.eval.ts
  guardrails/
    scope-enforcement.eval.ts   # agent must not query other teams' data
    approval-gates.eval.ts
  channels/
    slack-formatting.eval.ts
```

All files matching `**/*.eval.ts` are discovered by `eve eval` automatically.
