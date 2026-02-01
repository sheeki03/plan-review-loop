---
description: "Send your plan to GPT for iterative review. Loops until only minor issues remain."
argument-hint: "[path-to-plan-file]"
allowed-tools: ["Read", "Write", "Edit", "Glob", "Bash", "mcp__codex__codex"]
---

# Iterative Plan Review Loop

You are running an iterative plan review loop. GPT reviews the plan, you revise it, and the cycle repeats until the plan is clean.

---

## CRITICAL: Output Presentation Rules

These rules are NON-NEGOTIABLE. Every violation degrades the user experience. Read them carefully.

### Rule 1: ZERO raw GPT output
NEVER paste, quote, summarize, or paraphrase GPT's raw response in your text output. The user must NEVER see GPT's words directly. You MUST parse the structured response (VERDICT, ISSUES, OPEN QUESTIONS, SUMMARY) into your own formatted output using the templates below.

**What "parse" means:**
1. Extract the VERDICT string
2. Extract each issue's severity, description, and recommendation
3. Extract each open question and its recommendation
4. Extract the severity counts
5. Plug these values into the markdown template

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

### Rule 4: Use task tracking
Create a task list at the start to show progress. Update it as you go. This is the ONLY progress indicator — not narration text.

### Rule 5: NO inline issue summaries between iterations
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

Once found, output EXACTLY this and nothing else:

```
## Plan Review Loop

**Plan:** `{filename}`
**Starting review...**
```

---

## Step 2: Run the Review Loop

Read the plan reviewer prompt from `${CLAUDE_PLUGIN_ROOT}/prompts/plan-reviewer.md`.

For each iteration (no hard cap — keep going until the plan is clean):

### 2a. Send to GPT (silently)

Call `mcp__codex__codex` with the delegation prompt below. Output NOTHING before or after the call. No narration.

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
- Set VERDICT to REVISE if more than 4 MEDIUM/LOW issues remain
- Set VERDICT to ACCEPTABLE only if zero CRITICAL/HIGH issues AND 4 or fewer MEDIUM/LOW issues remain
- ALWAYS include an OPEN QUESTIONS section, even if verdict is ACCEPTABLE — surface anything ambiguous with your best recommendation

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

Parse GPT's response internally. Extract: VERDICT, each issue (severity + description + recommendation), each open question (question + recommendation), severity counts.

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

**IMPORTANT:** The `{description}` and `{recommendation}` values come from GPT's parsed output, but rewrite them in your own concise words. Do NOT copy GPT's phrasing verbatim. Keep each description to 1 sentence. Keep each recommendation to 1-2 sentences.

### 2c. Decide: Loop or Stop

**STOP if:**
- Zero CRITICAL and HIGH issues AND 4 or fewer MEDIUM/LOW issues remain

**CONTINUE if:**
- Any CRITICAL or HIGH issues exist, OR
- More than 4 MEDIUM/LOW issues exist

There is no max iteration cap. Keep looping until the plan is clean. If after 8 iterations the plan is still not clean, pause and ask the user whether to continue or stop.

### 2d. If Continuing: Revise the Plan (silently)

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

1. **NEVER output raw GPT text.** Parse it, rewrite it concisely, plug it into templates.
2. **NEVER narrate.** No "Let me...", "Now I'll...", "Continuing...", "Found X issues...". The templates ARE the output.
3. **NEVER add text between templates.** The sequence is: iteration template → transition template → next iteration template. Nothing else.
4. **NEVER silently skip issues.** Every CRITICAL/HIGH must be addressed before stopping.
5. **NEVER dump the plan content** back to the user in your output.
6. **Keep the plan's original structure.** Fix issues, don't rewrite from scratch.
7. **Each codex call is STATELESS.** Always include the full updated plan, not diffs.
8. **If you hit 8 iterations without converging,** pause and ask the user.
9. **ALWAYS include Open Questions** in every iteration and in the final report, even if there are none (say "No open questions.").
