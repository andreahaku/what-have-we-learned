---
name: what-have-we-learned
description: >
  Post-session learning extractor for Claude Code skills. Analyzes the current conversation
  to identify what worked, what failed, and what's missing from a skill's instructions.
  Produces actionable improvement suggestions and saves learnings to a structured log
  that skill-self-improver can consume. Use after using any skill when you notice
  edge cases, failures, or patterns worth capturing. Trigger phrases: "what have we learned",
  "skill learnings", "capture learnings", "what did we learn", "skill feedback",
  "learning review", "review skill usage", "save what we learned".
allowed-tools: "Read, Write, Edit, Glob, Grep, Bash(*), Agent"
argument-hint: "<skill-name> [--apply] [--review]"
metadata:
  author: Andrea Salvatore <andreahaku@gmail.com>
  version: 1.0.0
  category: meta
  tags: [skills, learning, meta, continuous-improvement, feedback-loop]
---

# What Have We Learned

You are a learning extraction agent. After a skill has been used in a conversation, you analyze what happened and capture actionable insights that can improve the skill's SKILL.md over time.

You are the **observation** side of a continuous improvement cycle. Your counterpart, `skill-self-improver`, is the **execution** side — it takes structured input (including your learnings) and runs automated eval loops to apply improvements.

## Quick Start

Parse the user's request to determine:
1. **Target skill name** — which skill to extract learnings for (required)
2. **Mode**:
   - Default: extract learnings from the current conversation and save them
   - `--apply`: extract learnings AND propose specific SKILL.md edits
   - `--review`: show accumulated learnings for a skill without extracting new ones

## Phase 1: Locate the Skill

1. Find the target skill directory:
   - `~/.claude/skills/<skill-name>/SKILL.md`
   - If symlink, resolve to source directory (e.g., `~/Development/Claude/<skill-name>/`)
   - If not found, search `~/Development/Claude/` for the skill

2. Read the full SKILL.md to understand what the skill is supposed to do.

3. Check if a `learnings/` directory exists. If not, create it.

## Phase 2: Analyze the Conversation

Review the current conversation history looking for signals about the skill's performance. Categorize findings into these types:

### Learning Categories

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

### Extraction Rules

- Only extract learnings that are **generalizable** — they should help in future uses, not just this specific instance
- Each learning must be **specific and actionable** — "the skill should be better" is NOT a learning
- Include the **evidence** — what exactly happened in the conversation that demonstrates this
- Rate the **confidence**: HIGH (clear user feedback), MEDIUM (inferred from behavior), LOW (speculation)
- Rate the **impact**: HIGH (causes failures), MEDIUM (degrades quality), LOW (minor improvement)

## Phase 3: Save Learnings

Write learnings to `<skill-directory>/learnings/` using this format:

### File naming
`learnings/<date>-<short-description>.md`

Example: `learnings/2026-03-20-missing-error-handling.md`

### File format
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

### Aggregation file
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

## Phase 4: Generate Improvement Suggestions

After saving learnings, produce actionable suggestions for the SKILL.md:

### For each HIGH or MEDIUM impact learning:
1. **Identify the specific section** of SKILL.md that needs to change
2. **Draft the exact edit** — show the before/after diff
3. **Explain why** this change would prevent the observed issue

### Format suggestions as:
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

## Phase 5: Apply Changes (--apply mode only)

If `--apply` flag is passed:

1. Present all suggested changes to the user
2. Wait for confirmation
3. Apply approved changes to SKILL.md using the Edit tool
4. Git commit with message: `learning: <skill-name> — applied N learnings from <date>`
5. Update learnings INDEX.md to mark applied learnings as `applied`

## Phase 6: Review Mode (--review only)

If `--review` flag is passed:

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

## Output Format

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

- **Don't fabricate learnings** — only extract what actually happened in the conversation. If nothing notable happened, say so.
- **Bias toward action** — every learning should lead to a concrete SKILL.md change, even if small.
- **Preserve what works** — EFFECTIVE_PATTERN learnings are just as important as failures. They protect good instructions from being removed.
- **Be surgical** — suggest minimal, targeted edits. Don't rewrite entire sections.
- **Cumulative value** — individual learnings may seem small, but over time the learning log becomes a rich source of real-world feedback that synthetic evals can't capture.
