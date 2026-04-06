---
date: 2026-04-02T19:02:33-00:00
researcher: claude-sonnet-4-6
git_commit: f23a54a
branch: main
repository: nanoclaw
topic: "Nanoclaw Personal Assistant (Gimli) Setup"
tags: [setup, docker, obsidian, terminal, claw, gimli]
status: complete
last_updated: 2026-04-02
last_updated_by: claude-sonnet-4-6
type: implementation_strategy
---

# Handoff: Nanoclaw "Gimli" Personal Assistant Setup

## Task(s)

Setting up nanoclaw as a personal assistant called **Gimli** for the user. Work is in-progress — `npm ci` has been run, but setup steps have not been completed yet.

**Goals (discussed, not yet implemented):**
- Gimli as the assistant name (not Andy)
- Docker via OrbStack (not Apple Containers — previous setup failed)
- Obsidian vault mounted into the container (`~/Documents/Obsidian Vault`)
- Asana MCP configured inside the container
- Simple terminal chat interface (like `claude` CLI) — NOT Telegram/WhatsApp

**Completed:**
- Cloned nanoclaw to `~/projects/nanoclaw`
- Ran `npm ci` successfully
- Removed all old Apple Container nanoclaw containers (9 removed)
- Cleaned up `~/.config/nanoclaw/` (deleted old config)
- Docker confirmed working via OrbStack

**In Progress / Not Started:**
- Running setup steps (timezone, environment, container build, groups, register, mounts, service)
- Building the terminal REPL wrapper around `claw`
- Configuring Obsidian vault mount
- Configuring Asana MCP in container
- Writing Gimli persona CLAUDE.md

## Critical References

- `~/projects/nanoclaw/.claude/skills/claw/scripts/claw` — the one-shot CLI tool; need REPL wrapper around it
- `~/projects/nanoclaw/groups/main/CLAUDE.md` — Gimli persona goes here
- `~/projects/nanoclaw/setup/index.ts` — setup entry point (run per-step via `--step` flag)

## Recent changes

No code changes made yet. Only:
- `~/projects/nanoclaw/` cloned fresh from upstream
- `npm ci` run

## Learnings

- **`claw` is one-shot, not a REPL.** It runs a container, sends a prompt, prints response, exits. It does pass back a `sessionId` to stderr (`[session: <id>]`). A thin Python or shell REPL wrapper that captures the session ID and passes it to the next invocation is needed — ~50 lines. This is what gives the `claude`-like terminal experience.
- **nanoclaw requires at least one channel** to start (`src/index.ts:693` — fatal exit if `channels.length === 0`). For the terminal-only use case with `claw`, nanoclaw's main process doesn't need to run — `claw` spawns containers directly. This is the right path: skip channels entirely and just use `claw` + REPL wrapper.
- **`/setup` Claude Code skill** orchestrates setup steps — it calls `npx tsx setup/index.ts --step <name>` for each step. The skill couldn't run because of `disable-model-invocation`. Steps must be run manually.
- **Setup steps can be run independently:** `npx tsx setup/index.ts --step timezone`, `--step environment`, `--step container`, `--step groups`, `--step register`, `--step mounts`, `--step service`, `--step verify`
- **Register step args needed:** `--jid`, `--name`, `--trigger`, `--folder`, `--channel`, `--no-trigger-required`, `--is-main`, `--assistant-name` (see `setup/register.ts:27-68`)
- **Obsidian vault is at:** `/Users/moollaza/Documents/Obsidian Vault` (confirmed from `~/Library/Application Support/obsidian/obsidian.json`)
- **Credentials:** User has Anthropic API key available. CC OAuth token is in `~/.claude/session-env/` but hook blocks reading it. Use API key in `.env`.
- **OrbStack Docker works** — `docker ps` confirmed.
- **Apple Container CLI** still installed at `/usr/local/bin/container` — `claw` auto-detects runtime, prefers `docker` over `container` (see `claw:71-81`).
- **`use-native-credential-proxy`** skill exists — simplifies credentials vs OneCLI. Worth applying before container build.

## Artifacts

- `/Users/moollaza/projects/nanoclaw/` — cloned repo, ready for setup
- `/Users/moollaza/projects/nanoclaw/.claude/skills/claw/scripts/claw` — CLI tool to wrap
- `/Users/moollaza/projects/nanoclaw/setup/register.ts` — register step implementation
- `/Users/moollaza/projects/nanoclaw/groups/main/CLAUDE.md` — template to customize for Gimli

## Action Items & Next Steps

1. **Run setup steps** (from inside `~/projects/nanoclaw/`):
   ```
   npx tsx setup/index.ts --step timezone
   npx tsx setup/index.ts --step environment
   npx tsx setup/index.ts --step container   # builds Docker image
   npx tsx setup/index.ts --step mounts
   ```

2. **Register main group** (no channel needed for claw-only use):
   ```
   npx tsx setup/index.ts --step register \
     --jid "cli:main" \
     --name "Gimli" \
     --folder "main" \
     --channel "cli" \
     --no-trigger-required \
     --is-main \
     --assistant-name "Gimli"
   ```

3. **Apply `use-native-credential-proxy` skill** (optional but simpler than OneCLI):
   Run `/use-native-credential-proxy` or check `.claude/skills/` for the script.

4. **Create `.env`** with API key:
   ```
   ANTHROPIC_API_KEY=sk-...
   ASSISTANT_NAME="Gimli"
   TZ=America/New_York   # or user's timezone
   ```

5. **Build the Gimli REPL wrapper** — a ~50-line Python script at `~/bin/gimli` that:
   - Reads session ID from `~/.gimli_session`
   - Calls `claw [-s <session_id>] "<prompt>"` 
   - Captures new session ID from stderr and saves it
   - Loops with a `You: ` / `Gimli: ` prompt
   - Supports `exit`/`quit`/Ctrl-C gracefully

6. **Configure Obsidian vault mount** — add to `~/.config/nanoclaw/mount-allowlist.json`:
   ```json
   {
     "allowedRoots": [
       { "path": "/Users/moollaza/Documents/Obsidian Vault", "allowReadWrite": true }
     ],
     "blockedPatterns": [],
     "nonMainReadOnly": true
   }
   ```
   Then configure the mount in the group's `containerConfig` or via the mounts setup step.

7. **Write Gimli persona** in `groups/main/CLAUDE.md` — personal assistant context, Obsidian vault usage, Asana awareness, user's goals/responsibilities.

8. **Configure Asana MCP inside container** — add to `container/agent-runner/` Claude config so the agent inside the container can access Asana.

9. **Test end-to-end**: `gimli` → should open REPL, first message spins up container, responds as Gimli.

## Other Notes

- The user does NOT want Telegram/WhatsApp/Discord — purely terminal
- The nanoclaw main process (the message loop) does NOT need to run for claw-only use. `claw` is standalone.
- Scheduled/proactive tasks (morning briefings, Asana checks) will require the main nanoclaw process eventually — but defer that until basic chat works
- User is on macOS (Darwin 25.3.0, arm64), using zsh, OrbStack for Docker
- Nanoclaw repo: `~/projects/nanoclaw` (not `~/nanoclaw` or `~/src/nanoclaw`)
- The `NANOCLAW_DIR` env var can be set so `claw` finds the right install location
- Skills available in the nanoclaw repo context: `/setup`, `/claw`, `/use-native-credential-proxy`, `/debug`, `/customize`
