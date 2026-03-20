---
description: "Send your plan to GPT for iterative review. Loops until only minor issues remain."
argument-hint: "[path-to-plan-file]"
allowed-tools: ["Read", "Write", "Edit", "Glob", "Bash", "AskUserQuestion", "mcp__codex__codex", "mcp__codex__codex-reply"]
---

# Iterative Plan Review Loop

You are running an iterative plan review loop. GPT reviews the plan, you revise it, and the cycle repeats until the plan is clean.

---

## CRITICAL: Output Presentation Rules

These rules are NON-NEGOTIABLE. Every violation degrades the user experience. Read them carefully.

### Rule 1: ZERO raw GPT output
NEVER paste, quote, summarize, or paraphrase GPT's raw response in your text output. The user must NEVER see GPT's words directly. You MUST parse the JSON response and plug extracted values into the templates below.

**What "parse" means:**
1. Parse the JSON response into an object
2. Extract `verdict`
3. Extract each issue's `severity`, `category`, `description`, and `recommendation`
4. Extract each open question's `question` and `recommendation`
5. Extract the `summary` counts
6. Plug these values into the markdown template

**FORBIDDEN patterns — NEVER output any of these:**
- "GPT found..." / "GPT says..." / "GPT's response..."
- "The review found..." followed by a prose summary
- "Key findings:" followed by a numbered list you wrote
- "Let me address..." followed by listing what GPT said
- Any bullet list that restates GPT's findings in your own words outside the template
- Quoting or block-quoting any part of GPT's response

### Rule 2: Use ONLY the exact templates
Your ENTIRE visible output for each phase must match the templates defined below. No extra text before, after, or between template sections. No commentary. No narration.

### Rule 3: ZERO narration
You must NEVER output any of these patterns:
- "Let me..." / "Now I'll..." / "I'll now..." / "Continuing..."
- "Reading the plan..." / "Sending to GPT..." / "Calling codex..."
- "Found X issues, let me fix them..."
- "The HIGH issues are: 1. ... 2. ... Let me address them."
- "Iteration N complete. GPT found..."
- Any explanation of what you're about to do or just did

**The ONLY text the user should see is the formatted templates.** Tool calls happen silently between template outputs.

### Rule 4: NO inline issue summaries between iterations
After presenting the iteration template, if the verdict is REVISE, you must:
1. Silently edit the plan file (no output)
2. Output ONLY the transition template (one line)
3. Silently call GPT again

Do NOT insert a summary of what you're about to fix, what GPT found, or what changes you're making. The iteration template already showed the issues. The transition template already summarizes what was fixed.

---

## Step 1: Find the Plan

Use this priority order:

**Priority 1: Explicit argument** — If the user provided a file path as an argument, use that.

**Priority 2: Current session's plan file** — If this session used plan mode, you already know the plan file path from the conversation context. Use that file. It lives in `~/.claude/plans/` with a random name.

**Priority 3: Most recent plan file** — Look in `~/.claude/plans/` and pick the most recently modified file. Show the user its name and first few lines, and ask them to confirm.

**Priority 4: Plan files in working directory** — Search for `plan.md`, `PLAN.md`, or any `*plan*.md`.

**If multiple plans found:** List them with path, date, and first line. Ask the user to pick using AskUserQuestion.

**If none found:** Ask the user for the path.

### Project Root Detection (after plan is found)

Determine `PROJECT_CWD` — the project root directory for codebase verification:

1. **Plan inside a project** (plan file path is NOT under `~/.claude/plans/`): resolve upward from the plan file's directory looking for root markers (`.git`, `package.json`, `Cargo.toml`, `go.mod`, `pyproject.toml`, `pom.xml`, `.claude`). Set `cwdConfident = true`.
2. **Plan from `~/.claude/plans/`**: resolve upward from the current session working directory looking for the same root markers. Set `cwdConfident = true` if markers found.
3. **No markers found in either case**: use the session working directory as `PROJECT_CWD`. Set `cwdConfident = false`.

### Preflight Check (advisory, not blocking)

Extract 2-3 file paths from the plan and Glob them against `{PROJECT_CWD}`. Track:
- `preflightHitCount`: how many paths matched
- `preflightHasUniqueDeepHit`: whether any hit is a deep/unique path (≥3 path segments, not generic like `package.json`, `index.ts`, `README.md`, `main.py`)

**Only ask user via AskUserQuestion if ALL of these are true:**
- `cwdConfident == true`
- The plan clearly references existing files (not mainly introducing new files)
- ALL extracted paths missed

Do not prompt the user if the plan mainly introduces new files.

### Codebase Verification Gate

```
codebaseVerificationEnabled = cwdConfident || preflightHitCount >= 2 || preflightHasUniqueDeepHit
```

- Always pass `cwd: {PROJECT_CWD}` to every Codex call.
- Include `CODEBASE_VERIFICATION: enabled` or `CODEBASE_VERIFICATION: disabled` in every prompt sent to Codex.
- When `disabled`: the reviewer performs a plan-only review and must NOT report missing files, missing symbols, or interface mismatches from the codebase.

Once found, output EXACTLY this and nothing else:

```
## Plan Review Loop

**Plan:** `{filename}`
**Starting review...**
```

---

## Step 2: Run the Review Loop

Read the plan reviewer prompt from `${CLAUDE_PLUGIN_ROOT}/prompts/plan-reviewer.md`.

Initialize state:
- `sessionActive = false`
- `threadId = null`
- `codebaseVerification = "enabled"` or `"disabled"` (from the gate above)

For each iteration (no hard cap — keep going until the plan is clean):

### 2a. Send to GPT (silently)

Output NOTHING before or after the call. No narration.

#### Iteration 1: Fresh call

Call `mcp__codex__codex` with:
- `prompt`: the first-call prompt (below)
- `developer-instructions`: contents of plan-reviewer.md
- `sandbox`: `read-only`
- `cwd`: `{PROJECT_CWD}`

After the response: extract `threadId` from the Codex **tool result envelope** (not from the JSON review body — the reviewer doesn't produce threadId). Then parse the response `content` as JSON for review findings. If `threadId` found in the envelope, store it and set `sessionActive = true`. If not found, `sessionActive = false`.

#### Iteration 2+ (session active — threadId available):

Call `mcp__codex__codex-reply` with:
- `threadId`: the stored threadId
- `prompt`: the follow-up prompt (below)

#### Iteration 2+ (fallback — no threadId):

Call `mcp__codex__codex` with ALL params:
- `prompt`: the follow-up prompt (below)
- `developer-instructions`: contents of plan-reviewer.md
- `sandbox`: `read-only`
- `cwd`: `{PROJECT_CWD}`

#### Session error recovery

If `codex-reply` returns an error: set `sessionActive = false`, retry as a fresh `mcp__codex__codex` call with all params, and attempt to extract a new `threadId` from the result.

#### First-call prompt:

```
TASK: Review this implementation plan. Find bugs, gaps, ambiguities, and anything that could go wrong.

CODEBASE_VERIFICATION: {enabled|disabled}
{IF enabled: "You have read access to the project codebase — verify file paths, symbols, and interfaces the plan references."}

<plan>
{FULL PLAN CONTENT}
</plan>

CONSTRAINTS:
- VERDICT is ACCEPTABLE only if zero CRITICAL/HIGH AND ≤4 MEDIUM/LOW issues remain. Otherwise REVISE.
- ALWAYS include OPEN QUESTIONS even if verdict is ACCEPTABLE.
```

#### Follow-up prompt (used for both resume and fallback):

```
Re-review this revised plan. Focus on whether previous issues were fixed and any new issues.

CODEBASE_VERIFICATION: {enabled|disabled — same value as iteration 1}
{IF enabled: "You have read access to the project codebase — verify file paths, symbols, and interfaces the plan references."}

<plan>
{FULL PLAN CONTENT}
</plan>

PREVIOUS ISSUES FIXED: {list}
REMAINING KNOWN ISSUES: {any, or "None"}

CONSTRAINTS:
- Do not re-raise issues that were fixed.
- VERDICT is ACCEPTABLE only if zero CRITICAL/HIGH AND ≤4 MEDIUM/LOW remain. Otherwise REVISE.
- ALWAYS include OPEN QUESTIONS even if verdict is ACCEPTABLE.
```

### 2b. Parse JSON and Present Results

Parse the JSON response. If parsing fails, follow the **JSON retry rules** below before presenting.

Extract from the parsed JSON: `verdict`, each issue (`severity`, `category`, `description`, `recommendation`), each open question (`question`, `recommendation`), and `summary` counts.

Then output this EXACT template and NOTHING ELSE — no text before it, no text after it:

```
---

### Iteration {N}

**Verdict:** {REVISE or ACCEPTABLE}

| Severity | Count |
|----------|-------|
| Critical | {n}   |
| High     | {n}   |
| Medium   | {n}   |
| Low      | {n}   |

**Issues found:**

- **[{SEVERITY}]** {description}
  > Fix: {recommendation}

- **[{SEVERITY}]** {description}
  > Fix: {recommendation}

{...repeat for each issue...}

**Open questions:**

- {question}
  > Recommendation: {answer}

{...repeat for each question. If none: "No open questions."}
```

**IMPORTANT:** The `{description}` and `{recommendation}` values come from the parsed JSON, but rewrite them in your own concise words. Do NOT copy GPT's phrasing verbatim. Keep each description to 1 sentence. Keep each recommendation to 1-2 sentences.

#### JSON Retry Rules

If the response is not valid JSON, retry ONCE:

**If `sessionActive` (threadId available):** Call `mcp__codex__codex-reply` with:
- `threadId`: stored threadId
- `prompt`: `"Your previous response was not valid JSON. Return ONLY the JSON object as specified."`

**If NOT sessionActive:** Rerun the full `mcp__codex__codex` call with all original parameters (`developer-instructions`, `sandbox`, `cwd`) and prepend this to the prompt: `"Your previous response was not valid JSON. Respond with ONLY the JSON object."` On success, extract new `threadId` and set `sessionActive = true`.

**If the second attempt also fails**, output the parse failure template and STOP:

```
---

## Review Failed

| | |
|---|---|
| **Iteration** | {N} |
| **Reason** | Codex returned invalid JSON twice |
| **Plan** | `{file path}` |

The review could not be parsed, so the loop stopped without applying further fixes.

---
```

### 2c. Decide: Loop or Stop

**STOP if:**
- Zero CRITICAL and HIGH issues AND 4 or fewer MEDIUM/LOW issues remain

**CONTINUE if:**
- Any CRITICAL or HIGH issues exist, OR
- More than 4 MEDIUM/LOW issues exist

There is no max iteration cap. Keep looping until the plan is clean. If after 8 iterations the plan is still not clean, pause and ask the user whether to continue or stop.

### 2d. If Continuing: Revise the Plan (silently)

**Which issues to fix — fix ALL of these, not just critical/high:**
- ALL CRITICAL issues — must fix
- ALL HIGH issues — must fix
- ALL MEDIUM issues — must fix (these are real problems, not nitpicks)
- LOW issues — fix if the fix is straightforward (1-2 line plan edit). Skip only if it would require significant plan restructuring.

**Do NOT cherry-pick only critical/high and leave medium issues for the next iteration.** Fixing only the top severity issues causes unnecessary extra iterations. GPT already filtered out nitpicks — if it rated something MEDIUM, it matters.

1. Apply fixes to the plan file using the Edit tool. Output NOTHING during this phase.
2. After all edits are done, output ONLY this transition template and NOTHING ELSE:

```
**Iteration {N} complete** — Fixed {X} issues ({list severities fixed, e.g. "2 critical, 1 high"}). Resubmitting...
```

3. Then immediately proceed to step 2a for the next iteration. No additional text.

**The full sequence the user sees per iteration is:**
```
---
### Iteration {N}
{severity table}
{issues list}
{open questions}

**Iteration {N} complete** — Fixed X issues (...). Resubmitting...
```

That's it. Nothing else between iterations.

---

## Step 3: Final Report

When the loop stops, output this EXACT template and NOTHING ELSE:

```
---

## Review Complete

| | |
|---|---|
| **Iterations** | {N} |
| **Verdict** | {ACCEPTABLE or REVISE} |
| **Issues fixed** | {total fixed across all iterations} |
| **Remaining** | {count remaining} |

### Remaining Issues

{For each remaining issue:}
- **[{SEVERITY}]** {description}
  > Recommendation: {recommendation}

{If no remaining issues: "None."}

### Open Questions

{ALWAYS include — list open questions with recommendations from the final iteration. If none: "No open questions."}

### Changes Made

**Iteration 1:** {2-3 bullet points of changes}
**Iteration 2:** {2-3 bullet points of changes}
...

**Plan updated at:** `{file path}`

---
```

If paused at 8 iterations with issues still open (and user chose to stop):

```
---

## Review Paused — User Stopped

**Warning:** {N} issues remain unresolved after {N} iterations.

### Unresolved Issues

{list the unresolved issues with severity}
- **[{SEVERITY}]** {description}
  > Recommendation: {recommendation}

### Open Questions

{list any remaining open questions with recommendations. If none: "No open questions."}

**Plan updated at:** `{file path}`

Manual review recommended before implementation.

---
```

---

## Important Rules

1. **NEVER output raw GPT text.** Parse the JSON, rewrite concisely, plug into templates.
2. **NEVER narrate.** No "Let me...", "Now I'll...", "Continuing...", "Found X issues...". The templates ARE the output.
3. **NEVER add text between templates.** The sequence is: iteration template → transition template → next iteration template. Nothing else.
4. **NEVER silently skip issues.** Every CRITICAL/HIGH must be addressed before stopping.
5. **NEVER dump the plan content** back to the user in your output.
6. **Keep the plan's original structure.** Fix issues, don't rewrite from scratch.
7. **Always include the full updated plan, not diffs.** When falling back to fresh calls, include `developer-instructions`, `sandbox`, and `cwd`.
8. **If you hit 8 iterations without converging,** pause and ask the user.
9. **ALWAYS include Open Questions** in every iteration and in the final report, even if there are none (say "No open questions.").
