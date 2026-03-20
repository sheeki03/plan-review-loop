# plan-review-loop

A Claude Code plugin that sends your implementation plans to GPT for iterative review. It loops — reviewing, fixing, and resubmitting — until only minor issues remain.

## How it works

```
/review-plan [path-to-plan]
```

1. **Finds your plan** — checks the argument, then session plan, then `~/.claude/plans/`, then `*plan*.md` in your working directory
2. **Detects the project root** — resolves upward from the plan or session cwd looking for `.git`, `package.json`, `go.mod`, etc.
3. **Runs a preflight check** — globs a few file paths from the plan to verify the codebase matches
4. **Sends to GPT via Codex** — delegates to `mcp__codex__codex` with the plan-reviewer system prompt
5. **Parses the JSON review** — extracts verdict, issues (with severity/category), open questions, and summary counts
6. **Presents formatted results** — renders a clean markdown template with severity table, issues, and recommendations
7. **Fixes the plan** — edits the plan file directly to address all CRITICAL, HIGH, and MEDIUM issues
8. **Loops** — resubmits via `mcp__codex__codex-reply` (session resume) until the verdict is ACCEPTABLE
9. **Final report** — summary of iterations, issues fixed, remaining items, and open questions

## Features

- **Session resume** — Iteration 1 uses a fresh Codex call; iteration 2+ resumes the thread via `codex-reply`, preserving reviewer context and reducing redundant work
- **Automatic fallback** — If session resume fails, falls back to a fresh call with full parameters
- **Codebase verification** — When the project root is confidently detected, GPT verifies file paths, function signatures, and interfaces the plan references (criterion 11)
- **JSON output** — The reviewer returns structured JSON, parsed into user-facing markdown templates. Malformed JSON triggers one automatic retry before failing gracefully
- **All-severity fixes** — Fixes CRITICAL, HIGH, and MEDIUM issues every iteration, not just the top severity. This minimizes unnecessary loop iterations
- **No iteration cap** — Loops until clean. Pauses at 8 iterations to ask if you want to continue

## Architecture

```
plugins/plan-review-loop/
├── .claude-plugin/
│   └── plugin.json            # Plugin manifest (v2.0.0)
├── commands/
│   └── review-plan.md         # Orchestrator — the full review loop logic
└── prompts/
    └── plan-reviewer.md       # GPT system prompt — review criteria + JSON output format

.claude-plugin/
└── marketplace.json           # Marketplace listing (v2.0.0)
```

### Flow

```
User ──▶ /review-plan ──▶ Find plan ──▶ Detect project root ──▶ Preflight check
                                                                       │
                          ┌────────────────────────────────────────────┘
                          ▼
                   Verification gate
                   (enabled/disabled)
                          │
                          ▼
              ┌──▶ Send to GPT (Codex) ──▶ Parse JSON ──▶ Present template
              │                                                   │
              │                                          ┌────────┴────────┐
              │                                     ACCEPTABLE          REVISE
              │                                          │                │
              │                                    Final report     Edit plan file
              │                                                          │
              └──────────────────────────────────────────────────────────┘
```

### Session resume

| Iteration | Method | Parameters |
|-----------|--------|------------|
| 1 | `mcp__codex__codex` | prompt + developer-instructions + sandbox + cwd |
| 2+ (active) | `mcp__codex__codex-reply` | threadId + follow-up prompt |
| 2+ (fallback) | `mcp__codex__codex` | full params (same as iteration 1) |

The `threadId` is extracted from the Codex tool result envelope after iteration 1. If `codex-reply` errors, the loop falls back to a fresh call and attempts to establish a new session.

### Codebase verification

Controlled by a gate that checks three conditions:

```
enabled = cwdConfident || preflightHitCount >= 2 || preflightHasUniqueDeepHit
```

When enabled, the reviewer (criterion 11) verifies:
- File paths exist in the codebase
- Function signatures and exports match what the plan assumes
- Named symbols are searchable

When disabled, the reviewer skips all codebase checks and does a plan-only review.

### Review criteria

1. Internal consistency
2. Bugs & logic errors
3. Completeness
4. Side effects & schema changes
5. Import & load-time safety
6. Blast radius control
7. Improvements
8. Open questions
9. Security & privacy
10. Dependencies & ordering
11. Codebase verification *(only when enabled)*

### Verdict rules

- **REVISE** — any CRITICAL or HIGH issues, or more than 4 MEDIUM/LOW
- **ACCEPTABLE** — zero CRITICAL/HIGH and at most 4 MEDIUM/LOW

## Prerequisites

- [Claude Code](https://claude.ai/claude-code) CLI
- [OpenAI Codex CLI](https://github.com/openai/codex) with MCP server running — provides `mcp__codex__codex` and `mcp__codex__codex-reply` tools

## Install

```bash
claude plugin install sheeki03/plan-review-loop
```

## Usage

```bash
# Review the plan from your current session
/review-plan

# Review a specific plan file
/review-plan ./plans/migration-plan.md

# Typical workflow
/plan Build a caching layer for the API    # write the plan
/review-plan                                # review and fix iteratively
```

## License

MIT
