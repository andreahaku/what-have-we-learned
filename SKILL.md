---
name: what-have-we-learned
description: >
  Post-session learning extractor for Claude Code skills. Analyzes the current conversation
  to identify what worked, what failed, and what's missing from a skill's instructions.
  Produces actionable improvement suggestions and saves learnings to a structured log
  that skill-self-improver can consume. Triggers automatically after any skill usage via
  CLAUDE.md rule — Claude performs a lightweight inline analysis and presents suggestions
  for user approval. Can also be invoked manually for deeper analysis or review.
  Trigger phrases: "what have we learned", "skill learnings", "capture learnings",
  "what did we learn", "skill feedback", "learning review", "review skill usage",
  "save what we learned".
allowed-tools: "Read, Write, Edit, Glob, Grep, Bash(*), Agent"
argument-hint: "[skill-name] [--apply] [--review]"
metadata:
  author: Andrea Salvatore <andreahaku@gmail.com>
  version: 2.0.0
  category: meta
  tags: [skills, learning, meta, continuous-improvement, feedback-loop]
---

# What Have We Learned

You are a learning extraction agent. After a skill has been used in a conversation, you analyze what happened and capture actionable insights that can improve the skill's SKILL.md over time.

You are the **observation** side of a continuous improvement cycle. Your counterpart, `skill-self-improver`, is the **execution** side — it takes structured input (including your learnings) and runs automated eval loops to apply improvements.

## Modes of Operation

This skill operates in two modes:

### Auto Mode (triggered by CLAUDE.md rule)
After any skill completes, Claude performs a **lightweight inline analysis** directly in the conversation — no sub-agent, no heavy processing. This is a quick scan that:
- Takes 30 seconds, not 5 minutes
- Outputs a compact block at the end of the response
- Only flags findings worth acting on — if nothing notable happened, outputs nothing
- **NEVER applies changes without explicit user approval**

### Manual Mode (invoked by user)
Full-depth analysis with persistent storage. Skill name is optional — if omitted, auto-detects from conversation. Three sub-modes:
1. **Default** (`/what-have-we-learned [skill-name]`): extract, save, and suggest
2. **Apply** (`/what-have-we-learned [skill-name] --apply`): extract, save, suggest, and apply approved changes
3. **Review** (`/what-have-we-learned [skill-name] --review`): show accumulated learnings

---

## Auto Mode: Inline Post-Skill Analysis

This is the primary mode. It runs automatically after every skill usage.

### When to trigger
After a skill completes (the Skill tool returns output), analyze the interaction **inline** — as part of your current response, not as a separate invocation.

### What to look for
Scan the conversation for these signals, in order of importance:

1. **User corrections** — the user said "no", "not like that", "actually...", or had to rephrase
2. **Unexpected failures** — the skill errored, produced wrong output, or missed a step
3. **Manual additions** — the user had to add something the skill should have included
4. **Smooth wins** — something worked particularly well that the skill doesn't explicitly instruct

### Decision: report or stay silent

**Stay silent if:**
- The skill worked as expected with no corrections
- Any issues were specific to this one case and not generalizable
- The findings are trivial (typos, minor formatting)

**Report if:**
- The user had to correct or supplement the skill's behavior
- A clear gap in the SKILL.md instructions was exposed
- A pattern emerged that would help in future uses
- Something worked exceptionally well and should be protected

### Auto Mode output format

When there ARE findings worth reporting, append this block at the end of your response:

```
---
**Skill Learning Detected** (`<skill-name>`)

<N> finding(s) from this session:

1. **<type>**: <one-line description>
   Fix: <specific SKILL.md change, 1-2 lines>

2. **<type>**: <one-line description>
   Fix: <specific SKILL.md change, 1-2 lines>

Save and apply? [y/n/skip]
```

Where `<type>` is one of: `Edge Case` | `Missing Step` | `Wrong Default` | `Good Pattern` | `Friction` | `Environment`

### User responses to auto mode

- **"y" or "yes"**: Save the learnings to `<skill-dir>/learnings/` AND apply the suggested SKILL.md changes. Commit with message `learning(<skill-name>): <brief description>`.
- **"n" or "no"**: Discard — don't save, don't apply. Move on.
- **"skip"**: Save the learnings to `<skill-dir>/learnings/` but do NOT apply changes to SKILL.md yet. They'll be available for `/what-have-we-learned <skill-name> --review` later.
- **"save only"**: Same as "skip" — save but don't apply.

### CRITICAL: Never auto-apply

**NEVER modify a skill's SKILL.md without explicit user approval.** The auto mode ONLY observes and suggests. The user decides what gets applied. This is non-negotiable.

---

## Manual Mode: Full Analysis

When the user explicitly invokes `/what-have-we-learned`, run the full analysis pipeline.

### Phase 0: Resolve Target Skill

Determine which skill to analyze based on what the user provided:

**If `<skill-name>` is given explicitly:**
Use it directly. Proceed to Phase 1.

**If NO skill name is given (`/what-have-we-learned` with no arguments, or with only flags like `--apply` or `--review`):**

1. **Scan the conversation** for Skill tool invocations. Look for any skill that was called via the Skill tool during this session.
2. **If exactly one skill was used:** use it as the target. Inform the user: `Detected skill: <skill-name>`.
3. **If multiple skills were used:** list them and ask the user which one to analyze:
   ```
   Multiple skills used in this session:
   1. approach
   2. ship
   3. pr

   Which skill would you like to analyze? (number or name, or "all" for all of them)
   ```
   If the user responds with `"all"`, run the full analysis pipeline sequentially for each skill.
4. **If no skills were used in this conversation:** inform the user and ask:
   ```
   No skills were used in this conversation. Which skill would you like to analyze?
   You can also use --review to check accumulated learnings for any skill.
   ```

### Phase 1: Locate the Skill

1. Find the target skill directory:
   - `~/.claude/skills/<skill-name>/SKILL.md`
   - If symlink, resolve to source directory (e.g., `~/Development/Claude/<skill-name>/`)
   - If not found, search `~/Development/Claude/` for the skill

2. Read the full SKILL.md to understand what the skill is supposed to do.

3. Check if a `learnings/` directory exists. If not, create it.

### Phase 2: Analyze the Conversation

Review the current conversation history looking for signals about the skill's performance. Categorize findings into these types:

#### Learning Categories

**1. EDGE_CASE** — A scenario the skill didn't handle well
- The user had to correct the skill's output
- The skill produced incorrect or incomplete results for a valid input
- A use case fell outside what the instructions anticipated

**2. MISSING_INSTRUCTION** — Something the skill should do but doesn't mention
- The user had to manually add something the skill should have included
- A step was needed that wasn't in the skill's workflow
- A tool or resource was needed that isn't listed

**3. WRONG_ASSUMPTION** — The skill's instructions assume something incorrect
- A default value or behavior doesn't match real-world usage
- An instruction leads to the wrong approach in practice
- A dependency or API works differently than the skill expects

**4. EFFECTIVE_PATTERN** — Something that worked particularly well
- An approach the skill took that the user confirmed or praised
- A pattern that produced high-quality output
- A workflow step that prevented errors

**5. FRICTION_POINT** — Something that slowed down or confused the workflow
- The user had to repeat themselves or clarify
- The skill asked unnecessary questions
- Output format didn't match what the user actually needed

**6. ENVIRONMENT_INSIGHT** — Something about the runtime environment that matters
- Tool availability or version constraints
- Platform-specific behavior (macOS, Linux, etc.)
- Dependency or configuration requirements

#### Extraction Rules

- Only extract learnings that are **generalizable** — they should help in future uses, not just this specific instance
- Each learning must be **specific and actionable** — "the skill should be better" is NOT a learning
- Include the **evidence** — what exactly happened in the conversation that demonstrates this
- Rate the **confidence**: HIGH (clear user feedback), MEDIUM (inferred from behavior), LOW (speculation)
- Rate the **impact**: HIGH (causes failures), MEDIUM (degrades quality), LOW (minor improvement)

### Phase 3: Save Learnings

Write learnings to `<skill-directory>/learnings/` using this format:

#### File naming
`learnings/<date>-<short-description>.md`

Example: `learnings/2026-03-20-missing-error-handling.md`

#### File format
```markdown
---
skill: <skill-name>
date: <YYYY-MM-DD>
source: conversation
categories: [EDGE_CASE, MISSING_INSTRUCTION]  # all categories found
impact: high | medium | low  # highest impact among findings
---

## Learnings from <date>

### 1. <Short title>
- **Category:** EDGE_CASE
- **Confidence:** HIGH
- **Impact:** HIGH
- **What happened:** <1-2 sentences describing what occurred>
- **Evidence:** <quote or paraphrase from conversation>
- **Suggested fix:** <specific change to SKILL.md>

### 2. <Short title>
...
```

#### Aggregation file
Also update (or create) `<skill-directory>/learnings/INDEX.md` — a summary of all learnings to date:

```markdown
# Learnings Index: <skill-name>

## Summary
- Total learnings: N
- High impact: N
- Unresolved: N (not yet applied to SKILL.md)

## By Category
- EDGE_CASE: N
- MISSING_INSTRUCTION: N
- WRONG_ASSUMPTION: N
- EFFECTIVE_PATTERN: N
- FRICTION_POINT: N
- ENVIRONMENT_INSIGHT: N

## Recent (last 5)
1. [<date>] <title> — <impact> — <status: new|applied|dismissed>
2. ...

## All Learnings
| Date | Title | Category | Impact | Status | File |
|------|-------|----------|--------|--------|------|
| ... | ... | ... | ... | ... | ... |
```

### Phase 3.5: Display Learnings to User

Before generating suggestions, **always show the extracted learnings to the user** in the output. For each learning, display:

```
### Learning 1: <Short title>
- **Category:** <EDGE_CASE | MISSING_INSTRUCTION | WRONG_ASSUMPTION | EFFECTIVE_PATTERN | FRICTION_POINT | ENVIRONMENT_INSIGHT>
- **Confidence:** <HIGH | MEDIUM | LOW>
- **Impact:** <HIGH | MEDIUM | LOW>
- **What happened:** <1-2 sentences>
- **Suggested fix:** <specific SKILL.md change>
```

This is NOT optional — the user must see the individual learnings with their ratings before the suggestions section. Do not skip to suggestions.

### Phase 4: Generate Improvement Suggestions

After saving learnings, produce actionable suggestions for the SKILL.md:

#### For each HIGH or MEDIUM impact learning:
1. **Identify the specific section** of SKILL.md that needs to change
2. **Draft the exact edit** — show the before/after diff
3. **Explain why** this change would prevent the observed issue

#### Format suggestions as:
```
## Suggested SKILL.md Changes

### Change 1: <title>
**Section:** <which part of SKILL.md>
**Reason:** <learning reference>
**Current:**
> <existing text>

**Proposed:**
> <new text>

**Impact:** Addresses learnings #1, #3
```

### Phase 5: Apply Changes (--apply mode or user approved)

1. Present all suggested changes to the user
2. **Wait for explicit confirmation** — NEVER auto-apply
3. Apply only the approved changes to SKILL.md using the Edit tool
4. Git commit with message: `learning(<skill-name>): applied N learnings from <date>`
5. Update learnings INDEX.md to mark applied learnings as `applied`

### Phase 6: Review Mode (--review only)

1. Read `learnings/INDEX.md` for the target skill
2. Read all learning files referenced as `new` (unresolved)
3. Present a summary:
   - Total accumulated learnings
   - Breakdown by category and impact
   - Top 3-5 unresolved high-impact learnings with suggested fixes
4. Ask if the user wants to:
   - Apply specific suggestions now
   - Dismiss any learnings that are no longer relevant
   - Run `skill-self-improver` with these learnings as input

---

## Integration with skill-self-improver

This skill produces structured output that skill-self-improver can consume:

1. **Learning files** in `<skill-dir>/learnings/` provide real-world failure cases
2. **Suggested changes** can seed mutation strategies (instead of random mutations)
3. **EFFECTIVE_PATTERN learnings** flag instructions that should NOT be modified during improvement loops
4. **INDEX.md** gives skill-self-improver a quick overview of known issues

When skill-self-improver runs, it should:
- Check for `learnings/INDEX.md` in the target skill
- Prioritize mutations that address HIGH impact unresolved learnings
- Protect instructions tagged as EFFECTIVE_PATTERN
- Generate eval assertions from EDGE_CASE learnings

## Output Format (Manual Mode)

Always end with a concise summary:

```
## Learning Summary: <skill-name>

Extracted: N learnings (H high, M medium, L low impact)
Saved to: <file-path>
Categories: EDGE_CASE(n), MISSING_INSTRUCTION(n), ...

Top actions:
1. <highest impact suggestion, 1 line>
2. <second highest, 1 line>
3. <third, 1 line>

Next steps:
- Run `/what-have-we-learned <skill-name> --apply` to apply suggestions
- Run `/skill-self-improver <skill-name>` to run automated improvement loop
```

## Important Notes

- **NEVER auto-apply changes** — all SKILL.md modifications require explicit user approval. This is the #1 rule.
- **Don't fabricate learnings** — only extract what actually happened in the conversation. If nothing notable happened, stay silent (auto mode) or say so (manual mode).
- **Bias toward action** — every learning should lead to a concrete SKILL.md change, even if small.
- **Preserve what works** — EFFECTIVE_PATTERN learnings are just as important as failures. They protect good instructions from being removed.
- **Be surgical** — suggest minimal, targeted edits. Don't rewrite entire sections.
- **Stay lightweight in auto mode** — the user didn't ask for a learning analysis; it's a bonus. Keep it brief and unobtrusive. If in doubt, stay silent.
- **Cumulative value** — individual learnings may seem small, but over time the learning log becomes a rich source of real-world feedback that synthetic evals can't capture.
