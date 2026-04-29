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

## Syncing with upstream

### Mental model

This fork is a tiny patchset (2 files: `Dockerfile` + `railway.toml`) sitting on top of upstream `NousResearch/hermes-agent`. Syncing means *replaying* upstream's new work and keeping your patches on top.

GitHub's "Sync fork" button only works when it can fast-forward — i.e., upstream hasn't touched anything near your changes. Once you've diverged (you have), do it locally instead.

### The local-vs-remote split (important)

Everything before `git push` is **local only** and doesn't touch Railway. Only `git push` triggers a Railway redeploy. So if a live process is running on the deployed instance (Telegram chat, autonomous loop, cron job), you can do the entire merge locally and **defer the push** until the process finishes.

Redeploy doesn't wipe `/opt/data` (the volume persists), but it does interrupt anything in-memory: the container gets SIGTERM, restarts, picks up disk state from where it was. Resumable conversations are fine; mid-write disk state is rare but worth waiting out.

### Safe merge workflow

```bash
# One-time: register the upstream remote
git remote add upstream https://github.com/NousResearch/hermes-agent.git

# Each sync:
git fetch upstream
git checkout main
git merge upstream/main
# ...resolve conflicts if any (see below)...
git diff upstream/main -- Dockerfile railway.toml   # verify exactly 2 divergences

# Only when ready (no live process to interrupt):
git push
```

### Conflict resolution

- **`Dockerfile` conflict:** Almost always means upstream touched something near the bottom of the file. Keep upstream's changes everywhere, but ensure the `VOLUME [ "/opt/data" ]` line stays deleted. If upstream removes `VOLUME` themselves, no conflict — accept the merge.
- **`railway.toml` conflict:** Won't happen because upstream doesn't have this file. If they ever add their own `railway.toml`, keep our `startCommand` (the full tini+entrypoint chain) over any simpler version they propose — the simple one doesn't work.

### Pre-push verification

```bash
git diff upstream/main -- Dockerfile railway.toml
```

Should show exactly two divergences: the missing `VOLUME` line in `Dockerfile`, and the entire `railway.toml`. Anything else means the merge went sideways — investigate before pushing.

### If something goes wrong after pushing

If a sync breaks the deployed instance:

- **Before pushing:** `git reset --hard ORIG_HEAD` undoes the merge cleanly.
- **After pushing:** `git revert -m 1 <merge-commit-sha>` and push the revert. Railway will redeploy the reverted state.

Railway will redeploy whatever's on `main`, so don't push a merge you haven't verified.

### Order of operations when you have feature branches in flight

If you also have a feature branch (e.g., `add-claude-md`) waiting to merge to `main`:

1. Merge `upstream/main` into `main` first.
2. Then merge or rebase your feature branch on top.
3. Then push.

Cleaner history, and conflicts (if any) surface against upstream first rather than getting tangled with your branch's changes.

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
