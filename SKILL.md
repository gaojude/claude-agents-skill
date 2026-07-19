---
name: claude-agents
description: Manage Claude Code background agents (the `claude agents` / FleetView list) — spawn new background jobs, list and summarize what each agent is doing, read their conversations, rename them, set their session color, and stop or delete them. Use when the user asks to spin up a background agent, check on / summarize their agents, clean up finished jobs, or change an agent's title or color.
---

# Managing `claude agents`

Background agents are sessions managed by the Claude Code daemon. There are two data sources.

1. The CLI, which is stable, so prefer it: `claude agents --json` lists live sessions (interactive and background) with `pid`, `id`, `name`, `status` (`busy`/`waiting`/`idle`), `state` (`working`/`blocked`/`done`), `sessionId`, `cwd`, and `startedAt`. Add `--all` to include completed background sessions. Works without a TTY.
2. The files, which are the source of truth and the only route for mutations: one directory per job under the jobs root. Resolve paths portably:

```bash
CONFIG_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
JOBS_DIR="$CONFIG_DIR/jobs"            # $JOBS_DIR/<8-char-id>/{state.json,timeline.jsonl,tmp/}
PROJECTS_DIR="$CONFIG_DIR/projects"    # transcripts live here (see below)
```

The `state.json` fields that matter: `state` (`working`/`blocked`/`done`), `name`, `nameSource` (`auto`/`user`), `detail` (last user message or status line), `output` (final `result` for done jobs), `sessionId`, `intent` (original prompt), `cwd`, `createdAt`/`updatedAt`.

A job's full conversation transcript is not in the job dir. It lives at:

```bash
# sessionId comes from state.json or `claude agents --json`
ls "$PROJECTS_DIR"/*/"<sessionId>".jsonl
```

If the user's `claude` is an alias with extra flags, call the real binary via `command claude ...` or `$(which -p claude)`.

Verified against Claude Code 2.1.215. The file formats are internal. If a recipe stops working on a newer version, re-verify against the binary before trusting it (e.g. `strings "$(readlink -f "$(command -v claude)")" | grep -i agentColor`).

## Spawn a new agent

```bash
cd /path/to/project && claude --bg "the task prompt"
```

Returns immediately; the job appears in the agents list with an auto-generated name. Useful flags: `--model <model>`, `--agent <agent-type>`, `--permission-mode <mode>`, `--add-dir <dir>`. The prompt should be self-contained, since the agent starts with no context beyond it and the project.

## List and summarize agents

Quick status: `claude agents --json` (add `--all` for completed). For a richer summary, read every `state.json`:

```bash
python3 - <<'EOF'
import json, glob, os
jobs_dir = os.path.join(os.environ.get("CLAUDE_CONFIG_DIR", os.path.expanduser("~/.claude")), "jobs")
for f in sorted(glob.glob(os.path.join(jobs_dir, "*", "state.json"))):
    j = json.load(open(f))
    out = j.get("output")
    if isinstance(out, dict):
        out = out.get("result") or out.get("text") or ""
    line = str(out or j.get("detail") or "").replace("\n", " ")[:120]
    print(f"{os.path.basename(os.path.dirname(f))}  {j.get('state','?'):8} {j.get('name','?')!r}  — {line}")
EOF
```

For "what has agent X actually been doing", read its transcript. It is JSONL; each line has a `type` of `user`/`assistant`/etc., with `message.content` as a string or a list of blocks (collect the `text` blocks). Summarize the tail; don't dump raw lines at the user. For long transcripts, delegate reading to a subagent and keep only the summary.

## Rename an agent (title)

There is no external CLI for this. Edit the job's `state.json`: set `name` to the new title and `nameSource` to `"user"` so auto-naming won't overwrite it. Use python's `json` (load, modify, dump) rather than sed.

## Set an agent's color

The in-session command is `/color <color>`. Palette: `red, blue, green, yellow, purple, orange, pink, cyan`; `default` (or `reset`) clears it; no argument picks randomly. Teammate sessions inside an agent team can't set their own color; their leader assigns it.

To set it for a session from outside, do what `/color` does internally and append one JSONL line to that session's transcript:

```bash
printf '{"type":"agent-color","agentColor":"cyan","sessionId":"%s"}\n' "$SESSION_ID" \
  >> "$PROJECTS_DIR/<project-slug>/$SESSION_ID.jsonl"
```

Locate the file by globbing for the sessionId as shown above. Colors render in the prompt bar and agents view; a live session may need to restart or refresh to pick up an externally appended color.

## Stop a running agent

`claude agents --json` gives the `pid`. Send SIGTERM (`kill <pid>`), then confirm it's gone from the list. Only kill jobs whose `kind` is `background`. Never kill an `interactive` session; that's a terminal the user has open.

## Delete and clean up agents

Removing a job from the list means deleting its directory:

```bash
rm -rf "$JOBS_DIR/<id>"
```

Rules:

- Always confirm with the user before deleting, listing exactly which jobs will go.
- If the job is still running (has a live pid in `claude agents --json`), stop it first.
- Deleting the job dir keeps the transcript in `$PROJECTS_DIR`, so the session remains resumable via `claude --resume <sessionId>`. Only delete the transcript too if the user explicitly wants the conversation gone.
- Job dirs without a `state.json` are stale leftovers, safe to offer for cleanup.
- Don't touch `$JOBS_DIR/pins.json` (UI pin state).
