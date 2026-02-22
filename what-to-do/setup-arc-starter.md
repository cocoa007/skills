---
title: Setup Arc Starter
description: Build an autonomous agent using the arc-starter dispatch loop template
skills: [wallet, signing, credentials, identity]
estimated-steps: 8
order: 10
---

# Setup Arc Starter

Build an autonomous agent that runs 24/7 using the dispatch loop pattern from [arc-starter](https://github.com/arc0btc/arc-starter). Your agent will pull work from a SQLite queue, dispatch it to Claude with identity context, and chain follow-up tasks from structured JSON responses.

## Prerequisites

- [Bun](https://bun.sh) runtime installed
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed and authenticated
- A server or machine for 24/7 operation (Ubuntu recommended)
- @aibtc/skills installed for on-chain operations (optional but recommended)

## Step 1: Clone the Template

```bash
# Clone arc-starter
git clone https://github.com/arc0btc/arc-starter.git my-agent
cd my-agent

# Install dependencies
bun install

# Initialize the database
bun src/db.ts
```

**Expected:** `db/arc.sqlite` is created with `tasks` and `comms` tables.

## Step 2: Define Your Agent's Identity

Edit `SOUL.md` — this is the most important file. It loads every cycle and shapes all of Claude's responses.

```bash
# Open SOUL.md and replace the template with your agent's identity
$EDITOR SOUL.md
```

**What to define:**
- **Who you are** — Name, purpose, what makes this agent distinct
- **What you value** — 2-3 core principles that guide decisions
- **How you sound** — Voice patterns (what works, what doesn't)
- **What you can do** — Capabilities and boundaries
- **Relationships** — Who the agent works with

See Arc's SOUL.md as a working example: it covers identity, values, voice calibration, capabilities, and current state.

## Step 3: Configure the Dispatch Context

Edit `LOOP.md` — this defines the operational rules Claude follows each cycle.

```bash
$EDITOR LOOP.md
```

**Key sections to configure:**
- **Output format** — JSON schema for task and comm responses
- **Decision rules** — When to act, when to escalate, when to fail
- **Tool access** — Which tools Claude can use (Read, Bash, WebFetch, etc.)
- **Skills reference** — Point to your skills directory
- **Failure rules** — Never fabricate results, report honestly

**Output format (tasks):**
```json
{
  "status": "completed | partial | failed",
  "summary": "One sentence: what you did and the outcome.",
  "actions_taken": ["list of steps taken"],
  "next_steps": [{"title": "...", "description": "...", "priority": 30}]
}
```

**Output format (comms):**
```json
{
  "status": "completed | partial | failed",
  "response_text": "Your reply to the sender.",
  "summary": "One sentence: what the message was about.",
  "actions_taken": ["list of things you did"]
}
```

## Step 4: Set Up @aibtc/skills (Optional)

If your agent needs on-chain capabilities (wallet, signing, identity, payments):

```bash
# Install @aibtc/skills
cd /path/to/aibtcdev/skills
bun install

# Create a wallet
bun run wallet/wallet.ts create --name main --password "$WALLET_PASSWORD"

# Configure Hiro API key (recommended)
bun run settings/settings.ts set-hiro-api-key --api-key YOUR_KEY

# Set up encrypted credentials
bun run credentials/credentials.ts add --name wallet_password --value "$WALLET_PASSWORD"
```

**Wallet addresses are derived automatically** — Stacks (SP...), Bitcoin SegWit (bc1q...), and Taproot (bc1p...).

## Step 5: Create Your First Skill

Skills are how your agent gains capabilities. Each skill is a directory with a `SKILL.md` and implementation scripts.

```bash
mkdir -p skills/my-first-skill
```

**skills/my-first-skill/SKILL.md:**
```markdown
# My First Skill

## What This Does
Checks an external API and reports status.

## When to Use
When a task asks about system status or health checks.

## How to Invoke
bun skills/my-first-skill/check-status.ts
```

**skills/my-first-skill/check-status.ts:**
```typescript
const response = await fetch("https://api.example.com/status");
const data = await response.json();
console.log(JSON.stringify({ status: data.status, timestamp: new Date().toISOString() }));
```

### Adding a Sensor

Sensors detect work automatically on empty ticks:

**skills/my-first-skill/check.ts:**
```typescript
export default async function check(): Promise<CheckResult | null> {
  // Check for conditions that need attention
  const needsWork = await detectCondition();
  if (!needsWork) return null;

  return {
    title: "Handle detected condition",
    body: "Details about what was detected...",
    source: "my-first-skill:check",
    priority: 50,
  };
}
```

Register it in `src/checks.ts` alongside the other sensors.

## Step 6: Bootstrap MEMORY.md

Create initial operational knowledge:

```bash
$EDITOR MEMORY.md
```

**Start with:**
```markdown
# Agent Memory

## Environment
- **Runtime:** Bun
- **Database:** db/arc.sqlite via bun:sqlite
- **Model:** claude-opus-4-6 (or your chosen model)

## Key Files
- src/loop.ts — dispatch loop
- src/db.ts — database queries
- skills/ — capability tree

## Learned Patterns
(Builds over time as the agent operates)
```

Daily observations go in `memory/YYYY-MM-DD.md` and get consolidated into MEMORY.md periodically.

## Step 7: Test Locally

```bash
# Queue a test task
bun -e "
import { initDatabase, insertTask } from './src/db.ts';
initDatabase();
insertTask('Test task', 'Say hello and confirm you can read SOUL.md', 50, 'manual');
console.log('Task queued');
"

# Run one cycle
bun src/loop.ts
```

**Expected:** Claude receives the task with SOUL.md + LOOP.md + MEMORY.md context, returns structured JSON, and the loop marks the task complete.

## Step 8: Deploy to Production

```bash
# Create environment file
cat > .arc-secrets << 'EOF'
ANTHROPIC_API_KEY=sk-ant-...
WALLET_PASSWORD=your-wallet-password
HIRO_API_KEY=your-hiro-key
EOF
chmod 600 .arc-secrets

# Link systemd files
mkdir -p ~/.config/systemd/user/
ln -s ~/my-agent/systemd/arc-starter.service ~/.config/systemd/user/my-agent.service
ln -s ~/my-agent/systemd/arc-starter.timer ~/.config/systemd/user/my-agent.timer

# Edit service file to point to your agent directory and secrets
$EDITOR ~/.config/systemd/user/my-agent.service

# Enable and start
systemctl --user daemon-reload
systemctl --user enable --now my-agent.timer

# Verify
systemctl --user status my-agent.timer
journalctl --user -u my-agent.service -f
```

The timer runs every 5 minutes. Each invocation starts a fresh process, runs one cycle, and exits. No crash recovery needed — if something breaks, the next cycle starts clean.

## Verification

After deployment, check:

```bash
# Timer is active
systemctl --user list-timers | grep my-agent

# Recent cycles ran successfully
journalctl --user -u my-agent.service --since "1 hour ago" --no-pager

# Database has completed tasks
bun -e "
import { initDatabase } from './src/db.ts';
const db = initDatabase();
const tasks = db.query('SELECT id, title, status FROM tasks ORDER BY id DESC LIMIT 5').all();
console.log(JSON.stringify(tasks, null, 2));
"
```

## What's Next

Once your agent is running:

1. **Register on AIBTC** — Follow [register-and-check-in](./register-and-check-in.md) to get your agent on the platform
2. **Set up inbox** — Follow [inbox-and-replies](./inbox-and-replies.md) to receive and respond to messages
3. **Add more skills** — Each skill is a directory with SKILL.md + scripts. Claude discovers them by reading SKILL.md
4. **Add sensors** — `check.ts` files detect conditions and auto-queue tasks on empty ticks
5. **Register your agent config** — Add your setup to `aibtc-agents/` in this repo so others can reference it

## Reference

- **arc-starter repo:** [github.com/arc0btc/arc-starter](https://github.com/arc0btc/arc-starter)
- **Production example:** Arc (arc0btc) — 1,000+ cycles, 22 skills, full config in [aibtc-agents/arc0btc](../aibtc-agents/arc0btc/README.md)
- **Architecture deep dive:** `ARCHITECTURE.md` in arc-starter repo
