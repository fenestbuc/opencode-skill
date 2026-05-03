---
name: opencode
description: "Delegate coding tasks to OpenCode CLI (autonomous AI coding agent)."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [Coding-Agent, OpenCode, AI-Agent, Code-Review, Refactoring, PTY, Automation]
    related_skills: [claude-code, codex, hermes-agent]
---

# OpenCode — Hermes Orchestration Guide

Delegate coding tasks to [OpenCode](https://opencode.ai) CLI via the Hermes terminal. OpenCode is an autonomous coding agent that can read files, write code, run shell commands, and manage git workflows. It supports multiple AI providers (OpenAI, Anthropic, Azure, Vercel AI Gateway, GitHub Copilot, Google, Perplexity, OpenCode Go) and can operate in print mode, interactive TUI, ACP server, or headless server modes.

## Prerequisites

- **Install:** OpenCode is installed at `~/.opencode/bin/opencode` (v1.14.33)
- **PATH note:** The binary is NOT on `PATH` by default. Always use the full path `~/.opencode/bin/opencode` or add `~/.opencode/bin` to PATH.
- **Auth:** Run `opencode providers login` for each provider you want to use. Credentials are stored in `~/.local/share/opencode/auth.json`.
- **Check providers:** `opencode providers list` shows configured credentials.
- **Check models:** `opencode models` lists all available models across all configured providers.
- **Version check:** `opencode --version`
- **Update:** `opencode upgrade` or `opencode upgrade <version>`
- **Health check:** `opencode doctor` or `opencode debug config`

## Global Paths

| Path | Purpose |
|------|---------|
| `~/.opencode/bin/opencode` | Binary |
| `~/.local/share/opencode/` | Data (auth.json, opencode.db, sessions, tool-output) |
| `~/.config/opencode/` | Config files |
| `~/.cache/opencode/` | Cache |
| `~/.local/state/opencode/` | State |
| `/tmp/opencode/` | Temporary files |

## Orchestration Modes

Hermes interacts with OpenCode in four modes. Choose based on the task.

### Mode 1: Print Mode (`opencode run`) — Non-Interactive (PREFERRED for most tasks)

Print mode runs a one-shot task and returns the result. No PTY needed. No interactive prompts.

```
terminal(command="~/.opencode/bin/opencode run 'Add error handling to all API calls in src/' --format json --dangerously-skip-permissions", workdir="/path/to/project", timeout=120)
```

**When to use print mode:**
- One-shot coding tasks (fix a bug, add a feature, refactor)
- CI/CD automation and scripting
- Structured data extraction via `--format json`
- Piped input processing (`cat file | opencode run "analyze this"`)
- Any task where you don't need multi-turn conversation

**Print mode flags:**
| Flag | Purpose |
|------|---------|
| `-m, --model` | Model in `provider/model` format (e.g., `opencode-go/kimi-k2.6`) |
| `--variant` | Reasoning effort: `high`, `max`, `minimal` (provider-specific) |
| `--thinking` | Show thinking blocks in output |
| `--dangerously-skip-permissions` | Auto-approve all tool use (no interactive prompts) |
| `--format json` | Output newline-delimited JSON events instead of formatted text |
| `-f, --file` | Attach file(s) to the message |
| `--title` | Set session title |
| `--agent` | Use a specific agent (e.g., `build`, `explore`) |
| `--attach` | Attach to a running opencode server instead of spawning |
| `--dir` | Directory to run in |
| `--port` | Port for local server (random if not specified) |
| `-c, --continue` | Continue last session |
| `-s, --session` | Resume specific session ID |
| `--fork` | Fork session when continuing |
| `--share` | Share the session |
| `--pure` | Run without external plugins (faster startup, eliminates plugin errors) |

### Mode 2: Interactive TUI via tmux — Multi-Turn Sessions

Interactive mode gives you a full conversational REPL. Requires tmux orchestration.

```
# Start a tmux session
terminal(command="tmux new-session -d -s opencode-work -x 140 -y 40")

# Launch OpenCode inside it
terminal(command="tmux send-keys -t opencode-work 'cd /path/to/project && ~/.opencode/bin/opencode' Enter")

# Wait for startup, then send your task
terminal(command="sleep 5 && tmux send-keys -t opencode-work 'Refactor the auth module to use JWT tokens' Enter")

# Monitor progress by capturing the pane
terminal(command="sleep 15 && tmux capture-pane -t opencode-work -p -S -50")

# Send follow-up tasks
terminal(command="tmux send-keys -t opencode-work 'Now add unit tests for the new JWT code' Enter")

# Exit when done
terminal(command="tmux send-keys -t opencode-work '/exit' Enter")
```

**When to use interactive mode:**
- Multi-turn iterative work (refactor → review → fix → test cycle)
- Tasks requiring human-in-the-loop decisions
- Exploratory coding sessions
- When you need to use slash commands

### Mode 3: ACP Server (`opencode acp`) — External Agent Connection

Start an Agent Client Protocol server that other agents can connect to.

```
# Start ACP server in background
terminal(command="~/.opencode/bin/opencode acp --port 8080", background=true)

# Attach from another process
terminal(command="~/.opencode/bin/opencode attach http://127.0.0.1:8080")
```

**ACP flags:**
| Flag | Purpose |
|------|---------|
| `--port` | Port to listen on (default: random) |
| `--hostname` | Hostname to bind (default: `127.0.0.1`) |
| `--mdns` | Enable mDNS discovery (sets hostname to `0.0.0.0`) |
| `--mdns-domain` | Custom mDNS domain (default: `opencode.local`) |
| `--cors` | Additional CORS domains |
| `--cwd` | Working directory |

### Mode 4: Headless Server (`opencode serve`)

Start a headless server without the TUI. Useful for API-like access.

```
terminal(command="~/.opencode/bin/opencode serve --port 8080", background=true)
```

## Print Mode Deep Dive

### JSON Output Format

Use `--format json` to get structured, machine-readable output:

```
terminal(command="~/.opencode/bin/opencode run 'Review auth.py for security issues' --format json --dangerously-skip-permissions", workdir="/project", timeout=120)
```

Returns newline-delimited JSON events:

| Event Type | Description |
|------------|-------------|
| `step_start` | Beginning of an agentic step |
| `tool_use` | Tool execution with input/output |
| `text` | Text response from the agent |
| `step_finish` | End of step with token/cost info |

**Key fields in `step_finish`:**
```json
{
  "type": "step_finish",
  "part": {
    "reason": "tool-calls|stop",
    "tokens": { "total": 9658, "input": 133, "output": 28, "reasoning": 25 },
    "cost": 0.000625068
  }
}
```

**Note:** Unlike Claude Code, OpenCode does not have a single final JSON result object. You must parse the event stream.

### Filtering JSON Output

Extract just the text responses:
```
~/.opencode/bin/opencode run "Explain X" --format json --dangerously-skip-permissions | \
  jq -r 'select(.type == "text") | .part.text'
```

Extract tool outputs:
```
~/.opencode/bin/opencode run "Run tests" --format json --dangerously-skip-permissions | \
  jq -r 'select(.type == "tool_use" and .part.tool == "bash") | .part.state.output'
```

### Piped Input

```
# Pipe a file for analysis
terminal(command="cat src/auth.py | ~/.opencode/bin/opencode run 'Review this code for bugs' --format json --dangerously-skip-permissions", timeout=60)

# Pipe command output
terminal(command="git diff HEAD~3 | ~/.opencode/bin/opencode run 'Summarize these changes' --format json --dangerously-skip-permissions", timeout=60)
```

### Session Continuation

```
# Continue last session
terminal(command="~/.opencode/bin/opencode run 'Continue refactoring' --continue --format json --dangerously-skip-permissions", workdir="/project", timeout=120)

# Resume specific session
terminal(command="~/.opencode/bin/opencode run 'What did we do last time?' --session ses_xxx --format json --dangerously-skip-permissions", timeout=60)

# Fork a session (new ID, keeps history)
terminal(command="~/.opencode/bin/opencode run 'Try a different approach' --session ses_xxx --fork --format json --dangerously-skip-permissions", timeout=120)
```

### Attaching to a Running Server

```
# Connect to an existing opencode server instead of spawning a new one
terminal(command="~/.opencode/bin/opencode run 'Check status' --attach http://127.0.0.1:8080 --format json", timeout=60)
```

## Interactive Session: Monitoring via tmux

### Reading the TUI Status
```
# Periodic capture to check if OpenCode is still working or waiting for input
terminal(command="tmux capture-pane -t opencode-work -p -S -10")
```

**Status indicators:**
- `❯` at bottom = waiting for input (done or asking a question)
- Tool execution lines = actively working (reading, writing, running commands)
- `●` or spinner = processing

### Dialog Handling

OpenCode may present permission prompts. In interactive mode, handle via tmux send-keys. For fully automated use, prefer print mode with `--dangerously-skip-permissions`.

## Provider & Model Management

### List Configured Providers
```
terminal(command="~/.opencode/bin/opencode providers list")
```

### Login to a Provider
```
terminal(command="~/.opencode/bin/opencode providers login <url>")
```

### Logout
```
terminal(command="~/.opencode/bin/opencode providers logout")
```

### List Available Models
```
terminal(command="~/.opencode/bin/opencode models")
```

### Select Model in Print Mode
```
terminal(command="~/.opencode/bin/opencode run 'Task' -m opencode-go/kimi-k2.6 --format json --dangerously-skip-permissions", timeout=120)
```

**Common providers:**
| Provider | Example Model |
|----------|---------------|
| `opencode-go` | `opencode-go/kimi-k2.6`, `opencode-go/deepseek-v4-pro` |
| `anthropic` | `anthropic/claude-sonnet-4-6`, `anthropic/claude-opus-4-6` |
| `openai` | `openai/gpt-5`, `openai/gpt-4o` |
| `azure` | `azure/gpt-4o`, `azure/claude-sonnet-4-6` |
| `github-copilot` | `github-copilot/claude-sonnet-4` |
| `google` | `google/gemini-2.5-pro` |
| `vercel` | `vercel/anthropic/claude-sonnet-4-6` |

## Session Management

### List Sessions
```
terminal(command="~/.opencode/bin/opencode session list")
```

### Delete a Session
```
terminal(command="~/.opencode/bin/opencode session delete <sessionID>")
```

### Export Session
```
terminal(command="~/.opencode/bin/opencode export <sessionID> --sanitize > /tmp/session.json")
```

**Critical:** `opencode export` prepends a log line (`Exporting session: ses_xxx`) before the JSON. Strip it before parsing:
```
terminal(command="~/.opencode/bin/opencode export <sessionID> --sanitize | tail -n +2 | jq '.info.id'")
```

### Import Session
```
terminal(command="~/.opencode/bin/opencode import /tmp/session.json")
```

**Note:** Sessions are stored in `~/.local/share/opencode/opencode.db` (SQLite). The `session list` command queries this database.

## Agent Management

OpenCode has built-in agents with different permission profiles:

| Agent | Type | Purpose |
|-------|------|---------|
| `build` | primary | General coding with full tool access |
| `compaction` | primary | Context compaction and summarization |
| `plan` | primary | Task planning (restricted to read + plan files) |
| `summary` | primary | Session summarization (read-only) |
| `title` | primary | Auto-generate session titles (read-only) |
| `explore` | subagent | Code exploration (read, grep, glob, bash) |
| `general` | subagent | General purpose assistant |

### List Agents
```
terminal(command="~/.opencode/bin/opencode agent list")
```

### Use a Specific Agent
```
terminal(command="~/.opencode/bin/opencode run 'Refactor auth.py' --agent build --format json --dangerously-skip-permissions", timeout=120)
```

## GitHub Workflow Integration

### Install GitHub Agent
```
terminal(command="~/.opencode/bin/opencode github install")
```

### Run GitHub Agent
```
terminal(command="~/.opencode/bin/opencode github run")
```

### Checkout and Review a PR
```
terminal(command="~/.opencode/bin/opencode pr 42")
```

This fetches and checks out the PR branch, then launches OpenCode.

## MCP Integration

### Add an MCP Server
```
terminal(command="~/.opencode/bin/opencode mcp add")
```

### List MCP Servers
```
terminal(command="~/.opencode/bin/opencode mcp list")
```

### Authenticate with OAuth MCP Server
```
terminal(command="~/.opencode/bin/opencode mcp auth <name>")
```

### Debug MCP Connection
```
terminal(command="~/.opencode/bin/opencode mcp debug <name>")
```

## Parallel OpenCode Instances

Run multiple independent tasks simultaneously:

```
# Task 1: Fix backend
terminal(command="tmux new-session -d -s task1 -x 140 -y 40 && tmux send-keys -t task1 'cd ~/project && ~/.opencode/bin/opencode run \"Fix the auth bug in src/auth.py\" --format json --dangerously-skip-permissions' Enter")

# Task 2: Write tests
terminal(command="tmux new-session -d -s task2 -x 140 -y 40 && tmux send-keys -t task2 'cd ~/project && ~/.opencode/bin/opencode run \"Write integration tests for the API endpoints\" --format json --dangerously-skip-permissions' Enter")

# Task 3: Update docs
terminal(command="tmux new-session -d -s task3 -x 140 -y 40 && tmux send-keys -t task3 'cd ~/project && ~/.opencode/bin/opencode run \"Update README.md with the new API endpoints\" --format json --dangerously-skip-permissions' Enter")

# Monitor all
terminal(command="sleep 30 && for s in task1 task2 task3; do echo '=== '$s' ==='; tmux capture-pane -t $s -p -S -5 2>/dev/null; done")
```

## Cost & Performance Tracking

### View Statistics
```
terminal(command="~/.opencode/bin/opencode stats")
```

### Filter Stats
```
terminal(command="~/.opencode/bin/opencode stats --days 7 --models 5")
```

### Cost Control Tips

1. **Use `--format json` sparingly** — it captures full event streams which can be verbose
2. **Set shell timeouts** — Hermes `terminal(timeout=120)` acts as a cost ceiling since OpenCode lacks built-in `--max-turns` or `--max-budget`
3. **Use specific agents** — `summary` and `title` agents are read-only and cheaper
4. **Model selection** — `opencode-go` models are often more cost-effective than premium providers
5. **Monitor via `stats`** — track spend across providers and models

## Environment Variables

| Variable | Effect |
|----------|--------|
| `OPENCODE_SERVER_PASSWORD` | Default password for server attach (`-p` flag) |

## Debugging

### Show Resolved Config
```
terminal(command="~/.opencode/bin/opencode debug config")
```

### Show All Paths
```
terminal(command="~/.opencode/bin/opencode debug paths")
```

### List All Projects
```
terminal(command="~/.opencode/bin/opencode debug scrap")
```

### List Available Skills
```
terminal(command="~/.opencode/bin/opencode debug skill")
```

### Agent Configuration
```
terminal(command="~/.opencode/bin/opencode debug agent build")
```

### Database Query
```
terminal(command="~/.opencode/bin/opencode db 'SELECT * FROM sessions LIMIT 5' --format json")
```

### Debug Without Plugins
Use `--pure` on any command to bypass external plugins. This eliminates plugin-related startup errors and reduces startup time:
```
terminal(command="~/.opencode/bin/opencode run 'Task' --pure --format json --dangerously-skip-permissions")
terminal(command="~/.opencode/bin/opencode debug config --pure")
```

## Pitfalls & Gotchas

1. **Binary is NOT on PATH** — always use `~/.opencode/bin/opencode` or add `~/.opencode/bin` to PATH first.
2. **No `--max-turns` or `--max-budget-usd`** — OpenCode lacks built-in cost/turn limits. Use Hermes `timeout` parameter as a proxy.
3. **JSON output is event-stream, not single result** — Unlike Claude Code's `--output-format json`, OpenCode returns newline-delimited events. You must parse the stream.
4. **`--dangerously-skip-permissions` is required for automation** — Without it, OpenCode may prompt for permission, causing the process to hang.
5. **Sessions are stored in SQLite** — `~/.local/share/opencode/opencode.db`. The `session list` command queries this DB.
6. **Provider auth is stored in `auth.json`** — never commit or expose `~/.local/share/opencode/auth.json`.
7. **Interactive mode requires tmux** — OpenCode is a TUI app. Use tmux for orchestration, not bare PTY.
8. **Model format is `provider/model`** — not just model name. Use `opencode models` to see valid values.
9. **`--variant` is provider-specific** — not all providers support `high`/`max`/`minimal`.
10. **Background tmux sessions persist** — always clean up with `tmux kill-session -t <name>` when done.
11. **`opencode pr` checks out the branch** — it modifies your working directory.
12. **ACP server defaults to random port** — specify `--port` if you need a fixed endpoint.
13. **`opencode export` prepends a log line** — output starts with `Exporting session: ses_xxx\n` before the JSON body. Pipe through `tail -n +2` before `jq`.

## Rules for Hermes Agents

1. **Prefer print mode (`run`) for single tasks** — cleaner, no dialog handling, structured output via `--format json`
2. **Always use full binary path** — `~/.opencode/bin/opencode` unless PATH has been configured
3. **Always set `--dangerously-skip-permissions` in print mode** — prevents permission prompt hangs
4. **Use tmux for multi-turn interactive work** — the only reliable way to orchestrate the TUI
5. **Always set `workdir`** — keep OpenCode focused on the right project directory
6. **Use `timeout` as cost control** — since OpenCode lacks `--max-turns`, rely on Hermes terminal timeouts
7. **Monitor tmux sessions** — use `tmux capture-pane -t <session> -p -S -50` to check progress
8. **Look for the `❯` prompt** — indicates OpenCode is waiting for input (done or asking a question)
9. **Clean up tmux sessions** — kill them when done to avoid resource leaks
10. **Report results to user** — after completion, summarize what OpenCode did and what changed
11. **Use `--format json` for automation** — parse the event stream for structured data extraction
12. **Use `--agent` for specialized tasks** — `build` for coding, `explore` for investigation, `summary` for read-only analysis
