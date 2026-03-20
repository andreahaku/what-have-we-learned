# What Have We Learned

A meta-skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that creates a continuous improvement feedback loop for your skills.

Every time you use a skill, you learn something — edge cases it doesn't handle, instructions that are missing, patterns that work well. This skill captures those insights before they're lost and turns them into actionable improvements.

## The Problem

Skills improve through two channels:
1. **Manual editing** — you notice something and update SKILL.md yourself
2. **Automated evals** — tools like [skill-self-improver](https://github.com/andreahaku/skill-self-improver) run synthetic tests

Both miss the most valuable signal: **what actually happens when a skill is used in real conversations**. Edge cases, user corrections, unexpected inputs, and workflow friction are observed in the moment but rarely captured.

## How It Works

### Auto Mode (default)

After installing, the skill works **automatically**. A CLAUDE.md rule triggers a lightweight inline analysis after every skill usage. If something notable happened — a user correction, a missing step, an unexpected failure — Claude appends a compact suggestion block:

```
---
**Skill Learning Detected** (`approach`)

2 finding(s) from this session:

1. **Missing Step**: Skill doesn't check for monorepo workspace dependencies before listing approaches
   Fix: Add "check package.json workspaces" to Step 1

2. **Good Pattern**: Presenting a compatibility matrix worked well for SDK comparisons
   Fix: Add as recommended format for SDK/library decisions

Save and apply? [y/n/skip]
```

**You decide what happens:**
- **`y`** — save the learning AND apply the SKILL.md change
- **`n`** — discard everything
- **`skip`** — save the learning for later, don't change SKILL.md yet

If nothing notable happened, the skill stays silent — no noise.

### Manual Mode

For deeper analysis or reviewing accumulated learnings:

```bash
# Full analysis of current conversation
/what-have-we-learned <skill-name>

# Full analysis + apply approved changes
/what-have-we-learned <skill-name> --apply

# Review all accumulated learnings for a skill
/what-have-we-learned <skill-name> --review
```

### Learning Categories

| Category | What it captures |
|---|---|
| `EDGE_CASE` | Scenarios the skill didn't handle well |
| `MISSING_INSTRUCTION` | Steps or behaviors the skill should include but doesn't |
| `WRONG_ASSUMPTION` | Instructions that assume something incorrect |
| `EFFECTIVE_PATTERN` | Approaches that worked particularly well (protects them from removal) |
| `FRICTION_POINT` | Things that slowed down or confused the workflow |
| `ENVIRONMENT_INSIGHT` | Runtime/platform-specific behaviors that matter |

## The Improvement Cycle

This skill is designed to work alongside [skill-self-improver](https://github.com/andreahaku/skill-self-improver) as two halves of a continuous improvement cycle:

```
              ┌─────────────────────────────┐
              │       Real Skill Usage       │
              └──────────────┬──────────────┘
                             │
                             ▼
              ┌─────────────────────────────┐
              │   what-have-we-learned      │
              │   (auto: observe & suggest) │
              └──────────────┬──────────────┘
                             │
                    user approves
                             │
                             ▼
              ┌─────────────────────────────┐
              │      learnings/ log         │
              │   (structured findings)     │
              └──────────────┬──────────────┘
                             │
                             ▼
              ┌─────────────────────────────┐
              │    skill-self-improver       │
              │  (automated eval & mutate)  │
              └──────────────┬──────────────┘
                             │
                             ▼
              ┌─────────────────────────────┐
              │     Improved SKILL.md       │
              └──────────────┬──────────────┘
                             │
                             └──────► back to Real Usage
```

When `skill-self-improver` runs, it reads the learnings log to:

- **Prioritize mutations** that address high-impact unresolved learnings
- **Generate eval assertions** from real-world edge cases (better than synthetic tests)
- **Protect effective patterns** — instructions tagged as `EFFECTIVE_PATTERN` are preserved during mutation loops

## Installation

### 1. Clone and symlink

```bash
git clone https://github.com/andreahaku/what-have-we-learned.git ~/Development/Claude/what-have-we-learned
ln -s ~/Development/Claude/what-have-we-learned ~/.claude/skills/what-have-we-learned
```

### 2. Enable auto-trigger

Add this section to your `~/.claude/CLAUDE.md`:

```markdown
## Skill Learning Loop

After any skill completes (via the Skill tool), perform a quick inline analysis of how the skill performed in this conversation. Follow the **Auto Mode** instructions in the `what-have-we-learned` skill:
- Scan for user corrections, unexpected failures, manual additions, or smooth wins
- If nothing notable happened, stay silent — do NOT output anything
- If there are actionable findings, append a compact "Skill Learning Detected" block at the end of your response
- **NEVER modify any skill's SKILL.md without explicit user approval** — only suggest, never auto-apply
- Keep it brief: 2-4 lines per finding, max. The user didn't ask for a report.
```

## Learning File Format

Learnings are saved as markdown files with YAML frontmatter in `<skill-dir>/learnings/`:

```markdown
---
skill: approach
date: 2026-03-20
source: conversation
categories: [EDGE_CASE, MISSING_INSTRUCTION]
impact: high
---

## Learnings from 2026-03-20

### 1. Skill doesn't handle monorepo context
- **Category:** EDGE_CASE
- **Confidence:** HIGH
- **Impact:** HIGH
- **What happened:** When evaluating approaches for a monorepo package, the skill didn't consider shared dependencies.
- **Evidence:** User had to manually point out that package A depends on package B's types.
- **Suggested fix:** Add a step to check for workspace dependencies before listing approaches.
```

An `INDEX.md` file in the same directory aggregates all learnings with status tracking (new/applied/dismissed).

## Design Principles

- **User is always in control** — changes to SKILL.md are never applied without explicit approval
- **Silent when nothing to report** — no noise after successful skill usage
- **Bias toward action** — every learning maps to a concrete SKILL.md edit
- **Preserve what works** — effective patterns are protected, not just failures captured
- **Cumulative value** — individual learnings are small, but the log becomes invaluable over time

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- Works best alongside [skill-self-improver](https://github.com/andreahaku/skill-self-improver)

## License

MIT
