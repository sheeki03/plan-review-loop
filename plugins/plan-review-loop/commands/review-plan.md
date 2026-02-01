---
description: "Send your plan to GPT for iterative review. Loops until only minor issues remain."
argument-hint: "[path-to-plan-file]"
allowed-tools: ["Read", "Write", "Edit", "Glob", "Bash", "mcp__codex__codex"]
---

# Iterative Plan Review Loop

You are running an iterative plan review loop. GPT will review the plan, you will revise it based on findings, and the cycle repeats until the plan is clean.

## How This Works

1. Read the plan file
2. Send it to GPT Plan Reviewer via `mcp__codex__codex`
3. Parse GPT's findings
4. If CRITICAL or HIGH issues exist → revise the plan and loop
5. If only MEDIUM/LOW remain (5 or fewer total) → stop and report

## Step 1: Find the Plan

Use this priority order:

### Priority 1: Explicit argument
If the user provided a file path as an argument, use that. Done.

### Priority 2: Current session's plan file
If this session used plan mode, you already know the plan file path from the conversation context (it's the file you were told to write your plan to). Use that file. It lives in `~/.claude/plans/` with a random name like `greedy-sprouting-wigderson.md`.

### Priority 3: Most recent plan file
If neither of the above, look in `~/.claude/plans/` and pick the most recently modified file. Show the user its name and first few lines, and ask them to confirm it's the right one.

### Priority 4: Plan files in working directory
Search the current working directory for `plan.md`, `PLAN.md`, or any `*plan*.md` files.

### If multiple plans found (and no explicit argument or session plan):
List ALL found plans with their path, last modified date, and first line. Ask the user to pick one using AskUserQuestion. Do NOT silently pick one.

### If no plan found:
Ask the user for the path.

## Step 2: Run the Review Loop

Read the plan reviewer prompt from `${CLAUDE_PLUGIN_ROOT}/prompts/plan-reviewer.md`.

For each iteration (max 5 iterations):

### 2a. Send to GPT

Call `mcp__codex__codex` with:
- `prompt`: Use the delegation format below, inserting the full plan content
- `developer-instructions`: The contents of the plan-reviewer prompt file
- `sandbox`: `read-only`

Delegation prompt template:
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

### 2b. Parse the Response

Extract from GPT's response:
- VERDICT (REVISE or ACCEPTABLE)
- List of issues with severities
- Open questions with recommendations
- Summary counts

### 2c. Decide: Loop or Stop

**STOP if:**
- VERDICT is ACCEPTABLE, OR
- Only MEDIUM and LOW issues remain with total count <= 5, OR
- Max iterations (5) reached

**CONTINUE if:**
- Any CRITICAL or HIGH issues exist
- More than 5 MEDIUM/LOW issues exist

### 2d. If Continuing: Revise the Plan

For each CRITICAL and HIGH issue:
- Apply GPT's recommendation to the plan
- Edit the plan file directly

For MEDIUM issues (if > 5 total remain):
- Apply the most impactful recommendations

Then notify the user:
```
Iteration {N} complete. Fixed {X} issues. Re-submitting for review...
```

## Step 3: Report Final Results

When the loop stops, output:

```
Plan Review Complete ({N} iterations)

Final Verdict: {VERDICT}

Remaining Issues ({count}):
{list remaining MEDIUM/LOW issues with recommendations}

Changes Made:
{summary of all revisions across iterations}

The plan has been updated at: {file path}
```

## Important Rules

- NEVER silently skip issues. Every CRITICAL/HIGH must be addressed.
- Show the user what changed after each iteration — don't just say "fixed".
- If you hit max iterations with CRITICAL issues still open, warn the user clearly.
- Keep the plan's original structure and intent. Fix issues, don't rewrite the plan from scratch.
- Each codex call is STATELESS. Always include the full updated plan, not diffs.
