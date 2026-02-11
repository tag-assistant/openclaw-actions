# 🦞 OpenClaw in GitHub Actions

> Run a persistent AI agent entirely inside GitHub Actions — no server required.

## What is this?

A self-restarting GitHub Actions workflow that runs [OpenClaw](https://github.com/openclaw/openclaw) as a persistent AI agent. The workflow chains itself before hitting the 6-hour job timeout, creating a continuous agent that can run indefinitely.

## How it works

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Chain #1    │────▶│  Chain #2    │────▶│  Chain #3    │────▶ ...
│  (5h 45m)   │     │  (5h 45m)   │     │  (5h 45m)   │
│              │     │              │     │              │
│ Install OC   │     │ Restore ws   │     │ Restore ws   │
│ Configure    │     │ Start gw     │     │ Start gw     │
│ Start gw     │     │ Run tasks    │     │ Run tasks    │
│ Run tasks    │     │ Save ws      │     │ Save ws      │
│ Save ws      │     │ Trigger #3   │     │ Trigger #4   │
│ Trigger #2   │     └──────────────┘     └──────────────┘
└──────────────┘
```

1. **Install** — Sets up Node 22 + OpenClaw CLI
2. **Restore** — Pulls workspace from cache (agent memory persists across chains)
3. **Configure** — Writes `openclaw.json` from secrets/template
4. **Run** — Starts the gateway, executes tasks, runs heartbeat loop
5. **Save** — Caches workspace + uploads logs as artifacts
6. **Chain** — Triggers the next workflow run before timeout

## Quick Start

### 1. Fork this repo

### 2. Add secrets

Go to **Settings → Secrets → Actions** and add:

| Secret | Description |
|--------|-------------|
| `OPENCLAW_MODEL_PROVIDER_KEY` | API key for your model provider (OpenAI, Anthropic, etc.) |
| `OPENCLAW_GATEWAY_TOKEN` | A random token for gateway auth (generate with `openssl rand -hex 32`) |
| `OPENCLAW_TELEGRAM_TOKEN` | *(Optional)* Telegram bot token for notifications |
| `OPENCLAW_TELEGRAM_CHAT_ID` | *(Optional)* Telegram chat ID for notifications |

### 3. Configure the model

Edit `config/openclaw.template.json` to set your provider:

```json
{
  "providers": {
    "openai": {
      "type": "openai",
      "apiKey": "${OPENCLAW_MODEL_PROVIDER_KEY}"
    }
  },
  "model": "openai/gpt-4o"
}
```

### 4. Run it

- **Manual:** Go to **Actions → 🦞 OpenClaw Agent → Run workflow**
  - Enter a task or leave blank for heartbeat mode
  - Enable/disable self-restart chain
  - Set max chain links (default: 12 = ~69 hours)
- **Scheduled:** Uncomment the cron schedule in the workflow

## Configuration

### Workflow Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `task` | *(empty)* | Task for the agent. Leave blank for heartbeat mode. |
| `chain` | `true` | Auto-restart before timeout |
| `max_chains` | `12` | Max chain restarts (0 = unlimited) |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_RUNTIME_SECONDS` | `20700` (5h 45m) | When to trigger restart |
| `HEARTBEAT_INTERVAL` | `300` (5 min) | Time between heartbeats |

## Architecture

### State Persistence

- **Workspace cache**: Agent memory, config, and state persist across chain links via `actions/cache`
- **Artifacts**: Logs and memory files uploaded after each chain for debugging
- **Key insight**: The agent wakes up with its memory intact, just like a normal OpenClaw restart

### Self-Restart Mechanism

The workflow uses `gh workflow run` to trigger itself before the 6-hour timeout:

```yaml
- name: Self-restart
  run: |
    gh workflow run openclaw.yml \
      -f chain_number="$((CURRENT + 1))" \
      -f max_chains="12"
```

This requires `actions: write` permission (already configured).

### Cost Estimation

On GitHub-hosted runners (ubuntu-latest):
- **Free tier:** 2,000 minutes/month → ~5.5 hours/day of agent time
- **Pro/Team:** 3,000 min/month → ~8.3 hours/day
- **Enterprise:** 50,000 min/month → ~138 hours/day (unlimited effectively)

Each chain link uses ~345 minutes. With `max_chains=12`, a full run is ~69 hours.

## Demo Ideas

This is great for demonstrating:

- 🤖 **AI-powered CI/CD** — Agent monitors repos, triages issues, reviews PRs
- 🔄 **Self-healing pipelines** — Agent detects failures and fixes them
- 📊 **Automated reporting** — Daily standups, metrics collection, status updates
- 🛡️ **Security monitoring** — Scan deps, check for vulnerabilities, open fix PRs
- 💬 **ChatOps** — Trigger tasks via issue comments, get results via Telegram

## Limitations

- **6-hour max per chain link** (GitHub Actions hosted runner limit)
- **Cache size limit**: 10GB per repo (workspace must stay under this)
- **Concurrent jobs**: Free tier allows 20 concurrent jobs (chain links are sequential)
- **No persistent network**: Each chain link gets a fresh runner (no long-lived connections)
- **Self-hosted runners**: Can bypass the 6-hour limit entirely (no chaining needed)

## License

MIT

---

Built with [OpenClaw](https://github.com/openclaw/openclaw) 🦞
