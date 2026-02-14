# claude-md-optimizer

A Claude Code skill that optimizes oversized CLAUDE.md files using progressive disclosure. Analyzes content tiers, detects encryption constraints, creates sub-documents, and rewrites the main file — with zero information loss.

## The Problem

CLAUDE.md files grow organically as projects mature. They accumulate troubleshooting guides, deployment commands, API references, and configuration examples alongside the essential rules needed every session. Since CLAUDE.md is loaded into every Claude Code conversation as system context, oversized files waste context window budget on content that's only needed occasionally.

Projects with 6+ months of development commonly reach 800-1,500+ lines.

## How It Works

The skill uses a 6-phase guided workflow:

1. **Analyze** — Read CLAUDE.md, parse sections, categorize each as Essential (every session), Reference (occasional), or Redundant (duplicates existing docs)
2. **Detect Constraints** — Check for git-crypt, SOPS, or age encryption; find safe unencrypted paths for sub-documents
3. **Plan** — Present a categorization table showing what stays inline vs. what gets extracted
4. **Review** — User approves the plan before any changes are made
5. **Execute** — Create sub-documents (verbatim extraction), rewrite CLAUDE.md with Sub-Documentation Table and progressive disclosure links
6. **Validate** — Diff total content before/after to confirm zero information loss

## Install

Copy this skill to your Claude Code skills directory:

```bash
git clone https://github.com/wrsmith108/claude-md-optimizer.git ~/.claude/skills/claude-md-optimizer
```

The skill is automatically available in all Claude Code sessions after installation.

## Usage

Trigger the skill by saying any of:

- "optimize CLAUDE.md"
- "reduce CLAUDE.md size"
- "CLAUDE.md too long"
- "apply progressive disclosure to CLAUDE.md"

The skill will analyze your file and present a plan for approval before making any changes.

## Skill Structure

```
claude-md-optimizer/
├── SKILL.md           # Entry point — workflow phases, essential rules, quick reference
├── analysis.md        # Content categorization methodology and heuristics
├── patterns.md        # 8 progressive disclosure patterns with decision tree
├── constraints.md     # Encryption detection, CI dependency scanning, safe path selection
├── validation.md      # Before/after diffing, failure modes, rollback procedure
└── README.md          # This file
```

## Progressive Disclosure Patterns

The skill applies these patterns when rewriting CLAUDE.md:

| Pattern | Use Case |
|---------|----------|
| **Sub-Documentation Table** | Navigation index at top of CLAUDE.md |
| **Quick-Reference + Link** | One essential command + link to full guide |
| **Compressed Table + Link** | Troubleshooting lookup table + link to diagnostics |
| **Summary + Link** | Brief description + link to comprehensive guide |
| **Inline Essential** | Content that must stay fully inline (no extraction) |
| **Split** | Keep essential summary inline, extract details |
| **Terse Agent Hint** | Compress multi-line code blocks to single-line command references |

## Key Safety Features

- **Guided Decision** — always presents the plan and waits for user approval before modifying files
- **Verbatim extraction** — content is moved, never summarized or paraphrased
- **Encryption-aware** — detects git-crypt, SOPS, and age; places sub-docs in unencrypted paths only
- **Chicken-and-egg protection** — encryption unlock instructions are force-classified as Essential and never extracted
- **CI dependency scanning** — detects scripts that regex-scan CLAUDE.md and keeps matched content inline
- **Zero-loss validation** — verifies total line count after optimization; aborts if content is missing

## Example Results

**Before**: 850 lines

**After**: 320 lines (62% reduction), 4 sub-documents (580 lines), zero information loss

## When NOT to Use

The skill will decline to optimize if:

- CLAUDE.md is under 300 lines
- More than 80% of content is Essential
- Less than 200 lines are extractable

## License

MIT
