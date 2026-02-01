---
description: "Send your plan to GPT for iterative review. Loops until only minor issues remain."
argument-hint: "[path-to-plan-file]"
allowed-tools: ["Read", "Write", "Edit", "Glob", "Bash", "mcp__codex__codex"]
---

# Iterative Plan Review Loop

You are running an iterative plan review loop. GPT reviews the plan, you revise it, and the cycle repeats until the plan is clean.

## CRITICAL: Output Presentation Rules

You MUST follow these rules for ALL output during the review loop. This is what the user sees — make it clean.

### Rule 1: Never dump raw GPT output
NEVER paste GPT's raw response to the user. Always parse it and present it in the formatted templates below.

### Rule 2: Use the exact output templates
Use the markdown templates defined in this file. Do not improvise formatting.

### Rule 3: Minimize noise
- Do NOT show the prompt you're sending to GPT
- Do NOT show "Reading file..." or "Calling codex..." narration
- Do NOT explain what you're about to do — just do it and show results
- Do NOT repeat the plan content back to the user

### Rule 4: Use task tracking
Create a task list at the start to show progress. Update it as you go.

---

## Step 1: Find the Plan

Use this priority order:

**Priority 1: Explicit argument** — If the user provided a file path as an argument, use that.

**Priority 2: Current session's plan file** — If this session used plan mode, you already know the plan file path from the conversation context. Use that file. It lives in `~/.claude/plans/` with a random name.

**Priority 3: Most recent plan file** — Look in `~/.claude/plans/` and pick the most recently modified file. Show the user its name and first few lines, and ask them to confirm.

**Priority 4: Plan files in working directory** — Search for `plan.md`, `PLAN.md`, or any `*plan*.md`.

**If multiple plans found:** List them with path, date, and first line. Ask the user to pick using AskUserQuestion.

**If none found:** Ask the user for the path.

Once found, output ONLY this:

```
## Plan Review Loop

**Plan:** `{filename}`
**Starting review...**
```

---

## Step 2: Run the Review Loop

Read the plan reviewer prompt from `${CLAUDE_PLUGIN_ROOT}/prompts/plan-reviewer.md`.

For each iteration (max 5):

### 2a. Send to GPT (silently)

Call `mcp__codex__codex` with the delegation prompt below. Do NOT show the prompt or mention the call.

Delegation prompt:
```
TASK: Review this implementation plan for bugs, improvements, open questions, and anything else that could go wrong. For any open questions, give your best recommendation — don't leave anything as "needs discussion".

EXPECTED OUTCOME: A structured review with severity-rated issues and actionable recommendations.

CONTEXT:
- Plan to review:

<plan>
{FULL PLAN CONTENT HERE}
</plan>

CONSTRAINTS:
- This is iteration {N} of the review loop
- Previous iterations found and fixed: {summary of previous fixes, or "N/A - first iteration"}

MUST DO:
- Rate every issue as CRITICAL, HIGH, MEDIUM, or LOW
- Provide a specific, actionable recommendation for every issue
- Provide your best recommendation for every open question
- Be specific and practical, not theoretical
- Include a SUMMARY with counts by severity
- Set VERDICT to REVISE if any CRITICAL or HIGH issues exist
- Set VERDICT to ACCEPTABLE if only 5 or fewer MEDIUM/LOW issues remain

MUST NOT DO:
- Leave open questions without a recommendation
- Flag style or formatting nitpicks
- Be vague ("improve error handling" — say exactly what and how)
- Repeat issues that were already fixed in previous iterations

OUTPUT FORMAT:
Follow the strict output format defined in your system instructions (VERDICT, ISSUES, OPEN QUESTIONS, SUMMARY).
```

Parameters:
- `developer-instructions`: Contents of the plan-reviewer prompt file
- `sandbox`: `read-only`

### 2b. Parse and Present Results

Parse GPT's response and present it using this EXACT template:

```
---

### Iteration {N} of 5

**Verdict:** {REVISE or ACCEPTABLE}

| Severity | Count |
|----------|-------|
| Critical | {n}   |
| High     | {n}   |
| Medium   | {n}   |
| Low      | {n}   |

**Issues found:**

{For each issue, format as:}
- **[{SEVERITY}]** {description}
  > Fix: {recommendation}

{If there are open questions:}

**Open questions:**

{For each question:}
- {question}
  > Recommendation: {answer}
```

### 2c. Decide: Loop or Stop

**STOP if:**
- VERDICT is ACCEPTABLE, OR
- Only MEDIUM and LOW issues remain with total count <= 5, OR
- Max iterations (5) reached

**CONTINUE if:**
- Any CRITICAL or HIGH issues exist
- More than 5 MEDIUM/LOW issues exist

### 2d. If Continuing: Revise the Plan

Apply fixes to the plan file silently (use Edit tool, don't show the edits inline).

Then output ONLY this transition:

```
---

**Iteration {N} complete** — Fixed {X} issues ({list severities fixed, e.g. "2 critical, 1 high"}). Resubmitting...

---
```

---

## Step 3: Final Report

When the loop stops, output this EXACT template:

```
---

## Review Complete

| | |
|---|---|
| **Iterations** | {N} |
| **Verdict** | {ACCEPTABLE or REVISE} |
| **Issues fixed** | {total fixed across all iterations} |
| **Remaining** | {count remaining} |

{If there are remaining issues:}

### Remaining Issues

{For each remaining issue:}
- **[{SEVERITY}]** {description}
  > Recommendation: {recommendation}

### Changes Made

{Bullet list of what was changed in the plan, grouped by iteration:}

**Iteration 1:** {2-3 bullet points of changes}
**Iteration 2:** {2-3 bullet points of changes}
...

**Plan updated at:** `{file path}`

---
```

If max iterations reached with CRITICAL/HIGH issues still open:

```
---

## Review Incomplete — Max Iterations Reached

**Warning:** {N} critical/high issues remain unresolved after 5 iterations.

{list the unresolved critical/high issues}

**Plan updated at:** `{file path}`

Manual review recommended before implementation.

---
```

---

## Important Rules

- NEVER silently skip issues. Every CRITICAL/HIGH must be addressed.
- NEVER dump raw GPT output. Always parse and format.
- NEVER narrate what you're doing ("Let me read the file...", "Now I'll call GPT..."). Just do it.
- If you hit max iterations with CRITICAL issues still open, warn clearly.
- Keep the plan's original structure and intent. Fix issues, don't rewrite from scratch.
- Each codex call is STATELESS. Always include the full updated plan, not diffs.
