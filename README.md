# What Have We Learned

A meta-skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that creates a continuous improvement feedback loop for your skills.

Every time you use a skill, you learn something — edge cases it doesn't handle, instructions that are missing, patterns that work well. This skill captures those insights before they're lost and turns them into actionable improvements.

## The Problem

Skills improve through two channels:
1. **Manual editing** — you notice something and update SKILL.md yourself
2. **Automated evals** — tools like [skill-self-improver](https://github.com/andreahaku/skill-self-improver) run synthetic tests

Both miss the most valuable signal: **what actually happens when a skill is used in real conversations**. Edge cases, user corrections, unexpected inputs, and workflow friction are observed in the moment but rarely captured.

## How It Works

`what-have-we-learned` bridges that gap. After using any skill, invoke it to extract and store learnings:

```
/what-have-we-learned <skill-name>
```

The skill analyzes the current conversation and:

1. **Extracts learnings** — categorized by type (edge cases, missing instructions, effective patterns, etc.)
2. **Saves structured logs** — to `<skill-dir>/learnings/` with metadata (confidence, impact, evidence)
3. **Maintains an index** — `learnings/INDEX.md` aggregates all findings over time
4. **Generates suggestions** — specific, surgical edits to the target skill's SKILL.md

### Learning Categories

| Category | What it captures |
|---|---|
| `EDGE_CASE` | Scenarios the skill didn't handle well |
| `MISSING_INSTRUCTION` | Steps or behaviors the skill should include but doesn't |
| `WRONG_ASSUMPTION` | Instructions that assume something incorrect |
| `EFFECTIVE_PATTERN` | Approaches that worked particularly well (protects them from removal) |
| `FRICTION_POINT` | Things that slowed down or confused the workflow |
| `ENVIRONMENT_INSIGHT` | Runtime/platform-specific behaviors that matter |

## Usage

### Extract learnings from the current conversation
```
/what-have-we-learned <skill-name>
```

### Extract and apply suggested changes to SKILL.md
```
/what-have-we-learned <skill-name> --apply
```

### Review accumulated learnings for a skill
```
/what-have-we-learned <skill-name> --review
```

## Integration with skill-self-improver

This skill is designed to work alongside [skill-self-improver](https://github.com/andreahaku/skill-self-improver) as two halves of a continuous improvement cycle:

```
Real usage → what-have-we-learned → learnings log
                                          ↓
                               skill-self-improver
                                          ↓
                              Improved SKILL.md → Real usage
```

When `skill-self-improver` runs, it reads the learnings log to:

- **Prioritize mutations** that address high-impact unresolved learnings
- **Generate eval assertions** from real-world edge cases (better than synthetic tests)
- **Protect effective patterns** — instructions tagged as `EFFECTIVE_PATTERN` are preserved during mutation loops

## Installation

Clone the repo and symlink into your Claude Code skills directory:

```bash
git clone https://github.com/andreahaku/what-have-we-learned.git ~/Development/Claude/what-have-we-learned
ln -s ~/Development/Claude/what-have-we-learned ~/.claude/skills/what-have-we-learned
```

## Learning File Format

Learnings are saved as markdown files with YAML frontmatter:

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

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- Works best alongside [skill-self-improver](https://github.com/andreahaku/skill-self-improver)

## License

MIT
