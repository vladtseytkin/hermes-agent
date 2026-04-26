# Fork-specific notes

This is a fork of [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent) deployed on Railway as a personal Telegram-only gateway. For general Hermes development, see `AGENTS.md` (upstream's dev guide). This file documents only the **deltas** from upstream and the **deployment shape** — both of which a future contributor (or future-you on a fresh machine) needs to know before merging upstream changes or touching the Dockerfile.

## Divergences from upstream

This fork modifies exactly **two files**. Preserve these on every upstream merge.

### 1. `Dockerfile` — `VOLUME` line removed

Upstream has `VOLUME [ "/opt/data" ]` near the bottom (between `ENV PATH=...` and `ENTRYPOINT [ ... ]`). This fork deletes that single line.

**Why:** Railway bans the `VOLUME` keyword in Dockerfiles — build fails immediately with `The 'VOLUME' keyword is banned in Dockerfiles`. Persistent storage is configured via Railway's dashboard volume system instead (mounted at `/opt/data` to match `HERMES_HOME`).

### 2. `railway.toml` — new file at repo root

Doesn't exist upstream:

```toml
[build]
builder = "DOCKERFILE"
dockerfilePath = "Dockerfile"

[deploy]
startCommand = "/usr/bin/tini -g -- /opt/hermes/docker/entrypoint.sh gateway run"
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 10
```

**Why the explicit start command:** Railway's `startCommand` *replaces* the Dockerfile `ENTRYPOINT` rather than appending to it. A simpler `startCommand = "gateway run"` fails with `The executable 'gateway' could not be found` because `gateway` is a `hermes` subcommand, not a binary on PATH. We spell out the full chain so:

- **`tini`** still reaps the MCP/git/bun stdio subprocesses (per the `Dockerfile:14` comment — without it, zombies accumulate under PID 1).
- **`docker/entrypoint.sh`** still activates the Python venv and bootstraps `/opt/data` (`.env`, `config.yaml`, skills sync).
- **`gateway run`** is then routed through the entrypoint's "wrap-as-`hermes`-subcommand" logic.

If upstream renames `docker/entrypoint.sh`, update the path in `railway.toml`.

## Resolving upstream-merge conflicts

```bash
git remote add upstream https://github.com/NousResearch/hermes-agent.git   # one-time
git fetch upstream
git merge upstream/main
```

- **`Dockerfile` conflict:** Almost always means upstream touched something near the bottom of the file. Keep upstream's changes everywhere, but ensure the `VOLUME [ "/opt/data" ]` line stays deleted. If upstream removes `VOLUME` themselves, no conflict — accept the merge.
- **`railway.toml` conflict:** Won't happen because upstream doesn't have this file. If they ever add their own `railway.toml`, keep our `startCommand` (the full tini+entrypoint chain) over any simpler version they propose — the simple one doesn't work.

**Sanity check after every merge:** `git diff upstream/main -- Dockerfile railway.toml` should show exactly these two divergences and nothing else.

## Railway deployment shape

- Project: `calm-insight` → env `production` → service `hermes-agent`
- Mode: **gateway-only** (interactive TUI is not deployed; it needs a PTY which Railway services don't have)
- Network: unexposed (no public URL); Telegram polling, so no inbound port
- Volume: mounted at `/opt/data` (= `HERMES_HOME`) — holds config, sessions, memories, skills, `.env`
- Build: Railway watches `main` on this fork → Dockerfile build → ~5–10 min first time

### Env vars (Railway → Variables tab)

Required for the gateway to do anything useful:

- `TELEGRAM_BOT_TOKEN` (from @BotFather)
- `TELEGRAM_ALLOWED_USERS` (comma-separated user IDs — without this, the bot is open to anyone)
- `OPENROUTER_API_KEY` *or* another model provider key (`OPENAI_API_KEY`, `NOUS_PORTAL_API_KEY`, etc.)

**Do NOT set `HERMES_UID`.** The entrypoint runs `usermod -u $HERMES_UID hermes` when it's set. UID `0` already belongs to root, so `usermod` fails with `UID '0' already exists`, `set -e` kills the script, container exits, Railway restarts — endless crash loop. Leave the variable unset; the entrypoint then keeps the hermes user at default UID 10000 and chowns the volume to it via gosu. The variable only matters for docker-compose setups where the container UID must align with the host's `~/.hermes` owner — Railway volumes have no host-side owner to align to.

The full env var reference is `.env.example` in the repo (large, well-commented).

## Interacting with the deployed agent

- **Day-to-day:** DM the Telegram bot. Slash commands (`/model`, `/personality`, `/skills`, `/usage`, `/help`) cover routine config.
- **Shell access:** `railway ssh` from a local machine after `railway link` (the dashboard SSH button isn't available on lower plans). Inside the container, run `hermes config show`, `hermes doctor`, `hermes model`, etc.
- **Logs:** Railway dashboard → service → Deployments → View logs.
- **Source changes:** Always edit locally → commit → push to `main` → Railway auto-rebuilds. Never edit source inside `railway ssh` — the next redeploy wipes it. Editing files on `/opt/data` *does* persist.
