# NotifAI — Agent Instructions

This file is kept in sync with `CLAUDE.md`. Any agent config, tool grants, or architectural changes must be reflected in both files.

## Product

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

## Key Files

- `VISION.md` — full architecture, event model, timeline
- `CLAUDE.md` — project config (synced with this file)
- `PROJECT.md` — living project memory, updated every session
- `src/index.ts` — agent entry point (main loop, REPL, single-shot)
- `src/lib.ts` — testable library (MessageStream, parseArgs, session, printing)
- `tests/lib.test.ts` — unit tests for lib.ts
- `docs/reviewer-accuracy.md` — per-reviewer accuracy tracking
- `vendor/nanoclaw/` — base platform reference (TypeScript, Claude Agent SDK)
- `vendor/openclaw/` — full-featured reference (tools, sessions, plugins)
- `vendor/nanobot/` — ultra-lightweight reference (Python)

## Tech Stack

- Node.js + TypeScript (ES2022/NodeNext/strict)
- Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`)
- Biome (lint + format)
- Vitest (tests)
- pre-commit (Biome → tsc → Vitest)

## Auth Model

- **Agent**: System Claude Code OAuth token. No separate API key — intentional design.
- **Commit often** — small, frequent commits trigger code review hooks.

## Development Rules

1. **Keep AGENTS.md in sync with CLAUDE.md** — any changes to either must be reflected in both.
2. **Meta capabilities, not specific tools** — grant agents broad meta capabilities (filesystem, shell, HTTP) not narrow integrations. The agent composes tools, not receives bespoke ones.
3. **Folder-level CLAUDE.md** — always create `CLAUDE.md` in new directories for quick indexing and context loading.
4. **Agent teams for parallelism** — always prioritize using agent teams (Task tool) to split work in parallel. Sequential is the last resort.
5. **Update PROJECT.md** — after completing work, update `PROJECT.md` with current state, decisions, and progress.
6. **Docs as code** — `PROJECT.md` captures what/how/why at the project level. Inline code comments must explain *why* (design choices, tradeoffs), not just *what*. Every non-obvious decision gets a comment.

## Architecture Guardrails (Planned)

- Classifier must stay stateless (single API call, no session)
- Secretary agent must be read-only (no send tools in toolset)
- Action bundles must be Zod-validated before writing to drafts/
- Tools must be meta capabilities, not specific integrations
- Approval endpoint is a separate code path from the agent

## Development Workflow

- **Pre-commit hooks**: Biome check → TypeScript type check → Vitest. All must pass before commit.
- **Code review**: On every `git commit`, 4 parallel reviewers (gpt-5.3-codex high, GLM-5, Claude Code, OpenCode Gemini 3.1 Pro low) review changes. Blocks with triage table.
- **Reviewer accuracy tracking**: After every triage, update `docs/reviewer-accuracy.md`.
- **Code simplifier**: After code review assessment, run `code-simplifier` subagent to tighten modified code.
- **Architect** (not wired): `architect-hook.sh` exists but is not configured in `settings.json`. Would run on Stop.
- **Project memory** (not wired): `project-memory-hook.sh` exists but is not configured in `settings.json`. Would run on UserPromptSubmit.
