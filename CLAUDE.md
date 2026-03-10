# NotifAI

Persistent personal agent running directly on the host with full access. Built on NanoClaw patterns (Claude Agent SDK, session persistence, MessageStream) without container isolation.

## Current State

Single-file agent loop with CLI interface. Session persistence via `~/.notifai/session.json`. No server, no classifier, no inbox pipeline yet — just the core agent.

## Architecture (Current)

```
CLI (REPL / single-shot)
  └─ Claude Agent SDK query()
       ├─ MessageStream (push-based async iterable, multi-turn)
       ├─ Session persistence (~/.notifai/session.json)
       └─ bypassPermissions (full host access, no gates)
```

## Architecture (Planned)

Dual-agent, event-driven:

- **Classifier** (Layer 1): Stateless OpenRouter call (Gemini 3.1 Flash Lite). Polls inbox/ every N seconds. Routes to human and/or secretary agent.
- **Secretary Agent** (Layer 2): Stateful Claude Agent SDK with persistent session. Produces action bundles. Read-only by architecture.
- **Inbox**: Append-only logs per channel. Webhooks land here. Classifier polls.
- **Policy layer**: Agent writes drafts. Separate approval endpoint executes.

See `VISION.md` for full vision, event model, and timeline.

## Tech Stack

- Node.js + TypeScript (ES2022/NodeNext/strict)
- Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`)
- Biome (lint + format)
- Vitest (tests)
- pre-commit (Biome → tsc → Vitest)

## Auth Model

- **Agent**: Uses the system's Claude Code OAuth token. No separate API key needed — the Claude Agent SDK authenticates via the same OAuth flow as the local Claude Code installation.
- **Commit often** — small, frequent commits trigger the code review hook which catches issues early.

## Project Index

```
VISION.md                   # Full vision, architecture, event model, timeline
CLAUDE.md                   # This file (keep in sync with AGENTS.md)
AGENTS.md                   # Agent instructions (synced with CLAUDE.md)
PROJECT.md                  # Living project memory — updated every session

src/
  index.ts                  # Agent entry point (main loop, REPL, single-shot)
  lib.ts                    # Testable library (MessageStream, parseArgs, session, printing)

tests/
  lib.test.ts               # Unit tests for lib.ts (10 tests)

docs/
  reviewer-accuracy.md      # Per-reviewer accuracy tracking (updated after every review triage)
  plans/                    # Design docs and implementation plans

vendor/                     # Reference implementations (git-ignored)
  nanoclaw/                 # Our base — lightweight agent (TypeScript, Claude Agent SDK)
  openclaw/                 # Full-featured persistent agent platform
  nanobot/                  # Ultra-lightweight agent (Python)

.claude/
  hooks/                    # Dev workflow hooks
    code-review-hook.sh     # PostToolUse — 4 parallel reviewers on commit (ACTIVE)
    architect-hook.sh       # Stop — Codex as product progression lead (NOT WIRED)
    project-memory-hook.sh  # UserPromptSubmit — PROJECT.md reminder (NOT WIRED)
    CLAUDE.md               # Hook-specific documentation
  settings.json             # Hook config
```

## CLI

```bash
tsx src/index.ts              # REPL mode (interactive)
tsx src/index.ts -m "query"   # Single-shot mode
tsx src/index.ts --new        # Fresh session (discard previous)
```

## Prior Work

- **NotifAI v1**: On-device Android classifier. Qwen3-0.6B, 94% folder accuracy, 83% priority, 1.3s.
- **Locus**: iOS agent with on-device Qwen 1.7B + EventKit
- **NanoClaw**: Base platform — channels, SQLite, scheduling, IPC, session persistence. Same SDK patterns, we skip the container layer.

## Development Rules

1. **Keep AGENTS.md in sync with CLAUDE.md** — any agent config, tool grants, or architectural changes must be reflected in both files.
2. **Meta capabilities, not specific tools** — grant agents broad meta capabilities (filesystem, shell, HTTP) not narrow integrations. The agent composes tools, not receives bespoke ones.
3. **Folder-level CLAUDE.md** — always create `CLAUDE.md` in new directories for quick indexing and context loading.
4. **Agent teams for parallelism** — always prioritize using agent teams (Task tool) to split work in parallel. Sequential is the last resort.
5. **Update PROJECT.md** — after completing work, update `PROJECT.md` with current state, decisions, and progress.
6. **Docs as code** — `PROJECT.md` captures what/how/why at the project level. Inline code comments must explain *why* (design choices, tradeoffs), not just *what*. Every non-obvious decision gets a comment.

## Development Workflow

- **Pre-commit hooks**: Biome check → TypeScript type check → Vitest. All must pass before commit.
- **Code review hook**: On every `git commit`, 4 parallel reviewers assess changes. Blocks with triage table.
  - **gpt-5.3-codex** — high reasoning, session persistence
  - **GLM-5** — Claude Code via z.ai, ephemeral
  - **Claude Code** — native Claude, ephemeral
  - **OpenCode** — Gemini 3.1 Pro via Antigravity, low variant, ephemeral
- **Reviewer accuracy tracking**: After every code review triage, update `docs/reviewer-accuracy.md` with each reviewer's finding counts (valid, deferred, invalid) and recompute accuracy percentages.
- **Code simplifier**: After code review assessment, run `code-simplifier` subagent to tighten modified code.
- **Architect hook** (not wired): `architect-hook.sh` exists but is not configured in `settings.json`. Would run on Stop.
- **Project memory hook** (not wired): `project-memory-hook.sh` exists but is not configured in `settings.json`. Would run on UserPromptSubmit.
- **CLI-first development**: All functionality starts as CLI commands before wiring into server/scheduler.
