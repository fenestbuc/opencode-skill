# OpenCode Skill for Hermes (v2)

A Hermes skill for delegating coding tasks to [OpenCode](https://opencode.ai), the autonomous AI coding agent CLI. This skill enables Hermes agents to orchestrate OpenCode in print mode, interactive TUI sessions, ACP server mode, and headless server mode.

## What It Does

- **Print mode delegation** (`opencode run`) — one-shot coding tasks with structured JSON output
- **Interactive TUI orchestration** — multi-turn sessions via tmux
- **ACP server integration** — spawn and attach to OpenCode via Agent Client Protocol
- **Multi-provider support** — OpenAI, Anthropic, Azure, Vercel AI Gateway, GitHub Copilot, Google, Perplexity, OpenCode Go
- **Session management** — resume, fork, export, import
- **GitHub workflow** — PR checkout and review via `opencode pr`
- **Native subagent delegation** — integration with Hermes `delegate_task` for parallel execution
- **Context management** — explicit handling of the 262K token context cap for models like `kimi-k2.6`

## Installation

1. Install OpenCode: https://opencode.ai
2. Copy `SKILL.md` into your Hermes skills directory:
   ```bash
   mkdir -p ~/.hermes/skills/autonomous-ai-agents/opencode
   cp SKILL.md ~/.hermes/skills/autonomous-ai-agents/opencode/SKILL.md
   ```
3. Verify: `hermes skills list | grep opencode`

## Quick Start

```bash
# One-shot task (print mode)
opencode run "Refactor auth.py to use JWT" -m opencode-go/kimi-k2.6 --format json --dangerously-skip-permissions

# Interactive session
tmux new-session -d -s opencode-work
opencode

# ACP server
opencode acp --port 8080
opencode attach http://127.0.0.1:8080
```

## Requirements

- [OpenCode CLI](https://opencode.ai) v1.14.33+
- Hermes Agent with skill support
- tmux (for interactive mode orchestration)
- jq (for JSON output filtering)

## License

MIT. See [LICENSE](./LICENSE).
