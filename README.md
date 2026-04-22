# claude-md-optimizer

A Claude Code skill that optimizes oversized agent instruction files using progressive disclosure. Handles **CLAUDE.md** (Claude Code), **AGENTS.md** (OpenAI Codex / GitHub Copilot agents), and **copilot-instructions.md** (GitHub Copilot). Auto-detects all three formats, analyzes content tiers, creates sub-documents using each format's native mechanism, and rewrites the main file — with zero information loss.

## The Problem

Agent instruction files grow organically as projects mature. They accumulate troubleshooting guides, deployment commands, API references, and configuration examples alongside the essential rules needed every session. Since these files are loaded into every AI session as context, oversized files waste the context window on content that's only needed occasionally — and for CLAUDE.md, Anthropic explicitly warns that adherence degrades above 200 lines.

Projects with 6+ months of development commonly reach 800–1,500+ lines.

## Supported Formats

| Format | Primary Location | Target Size | Native Sub-Doc Mechanism |
|--------|-----------------|-------------|--------------------------|
| CLAUDE.md | `./CLAUDE.md` or `./.claude/CLAUDE.md` | <200 lines | `@import` syntax; `.claude/rules/*.md` |
| AGENTS.md | `./AGENTS.md` (hierarchical) | <32KB combined | Nested `AGENTS.md` in subdirectories |
| copilot-instructions.md | `.github/copilot-instructions.md` | ~100 lines | `.github/instructions/NAME.instructions.md` |

## How It Works

The skill uses a 6-phase guided workflow:

1. **Detect & Analyze** — Auto-scan the repo for all three file types; report sizes vs. format targets; parse and categorize sections as Essential, Reference, or Redundant
2. **Detect Constraints** — Check for git-crypt/SOPS/age encryption; identify format-native sub-doc mechanisms; scan for CI scripts that regex-scan the file
3. **Plan** — Present a categorization table showing what stays inline vs. what gets extracted (and where)
4. **Review** — User approves the plan before any changes are made
5. **Execute** — Create sub-documents using the format's native mechanism; rewrite the main file with rich inline synopses and progressive disclosure links
6. **Validate** — Diff total content before/after to confirm zero information loss; verify format target is met

## Install

Copy this skill to your Claude Code skills directory:

```bash
git clone https://github.com/aronDisi/agents-md-optimizer.git ~/.agents/skills/claude-md-optimizer
```

The skill is automatically available in all Claude Code sessions after installation.

## Usage

Trigger the skill by saying any of:

- "optimize CLAUDE.md"
- "AGENTS.md is too long"
- "reduce copilot-instructions.md"
- "agent instructions file too large"
- "apply progressive disclosure"

The skill will detect your instruction files, report their sizes, and present a plan for approval before making any changes.

## Skill Structure

```
claude-md-optimizer/
├── SKILL.md           # Entry point — workflow phases, supported formats, essential rules
├── analysis.md        # Content categorization methodology, format thresholds, extraction strategies
├── patterns.md        # 13 progressive disclosure patterns including @import, .claude/rules/, path-scoped
├── constraints.md     # Encryption detection, AGENTS.md hierarchy, copilot path constraints, CI scanning
├── validation.md      # Per-format pass criteria, before/after diffing, failure modes, rollback
└── README.md          # This file
```

## Progressive Disclosure Patterns

| Pattern | Use Case |
|---------|----------|
| **Sub-Documentation Table** | Optional reference index — place at *end* of file, not top |
| **Rich Abstract + Link** | 3–5 sentence synopsis with concrete facts + link; agent answers most questions without opening the sub-doc |
| **Quick-Reference + Link** | One essential command + link to full guide |
| **Compressed Table + Link** | Troubleshooting lookup table + link to diagnostics |
| **Summary + Link** | Brief description + link to comprehensive guide |
| **Inline Essential** | Content that must stay fully inline |
| **Split** | Keep essential summary inline, extract details |
| **Terse Agent Hint** | Compress multi-line code blocks to single-line command references |
| **@Import Chain** | *(CLAUDE.md)* Reference files that load at session start |
| **.claude/rules/ Migration** | *(CLAUDE.md)* Move rules to topic files with optional path scoping |
| **Path-Scoped Instructions** | *(copilot-instructions.md)* Extract by language/framework/directory |
| **Contextual Reference** | *(AGENTS.md, copilot)* "For reason, see .agents/instructions/ref.md" inline link |

## The Rich Abstract Pattern: Why It Works

The central insight behind this skill is that **how you write a reference matters as much as whether you extract it**.

A thin link like "For testing details, see docs/testing.md" tells an agent a file exists but not whether it's relevant to the current task. An agent facing an ambiguous question will follow all such links to be thorough.

A rich abstract like this:

> Tests use **Vitest** (not Jest). Test files co-located with source, `.unit.spec.ts` suffix. Minimum **87% line coverage** enforced in CI for `src/services/` — failing blocks merge. Mock with `vi.mock()`. For snapshot testing and full mock patterns, see [docs/testing.md](docs/testing.md).

…gives the agent enough concrete facts (framework, naming convention, threshold) to answer the most common questions without opening the file at all. The link becomes a trapdoor, only followed when the synopsis is genuinely insufficient.

### Measured results

We ran a controlled usage eval comparing four strategies across focused and open-ended tasks (fixtures in `evals/fixtures/usage-eval-*/`). Each task asked an agent to complete a realistic developer request using an optimized instruction file with 5 sub-documents. We counted how many sub-doc files the agent actually opened:

| Strategy | Focused task | Ambiguous task |
|---|---|---|
| Thin inline ref ("For X, see Y") | 2 files | 5 files (all) |
| Lazy frontmatter on sub-docs only | 2 files | 5 files (all) |
| **Rich abstract + link** | **1 file (main only)** | **2 files** |
| Rich abstract + lazy frontmatter | 1 file | 2 files |

**Rich abstracts eliminated sub-doc reads entirely on focused tasks** — the synopsis was self-sufficient. On open-ended tasks, reads dropped from 5 to 2. Lazy frontmatter (`loading_strategy: lazy` + `description:` on sub-doc files) added no measurable reduction on top of a rich synopsis.

The one remaining sub-doc read on open-ended tasks (docker setup) makes sense: "first-time local setup" legitimately requires step-by-step commands that a synopsis can't fully substitute.

### What this means for context efficiency

Research into platform-level lazy loading (Claude Code, GitHub Copilot, Cursor) found that **instruction file content is loaded eagerly at session start** — the agent doesn't selectively load based on task scope. `loading_strategy: lazy` in copilot frontmatter appears unimplemented or aspirational as of 2026. The only proven platform-level lazy loading is Claude Code's `ToolSearch` for MCP tool schemas (85% token reduction), which doesn't apply to instruction files.

This means the savings from progressive disclosure come from two distinct places:
1. **Session startup**: reducing the main file's line count (fewer tokens loaded at startup)
2. **During session**: reducing the number of sub-doc files the agent explicitly reads (fewer tool calls)

Rich abstracts address both: a shorter main file at startup, and fewer file reads during the session.

## Key Safety Features

- **Guided Decision** — always presents the plan and waits for user approval before modifying files
- **Verbatim extraction** — content is moved, never summarized or paraphrased
- **Encryption-aware** — detects git-crypt, SOPS, and age; places sub-docs in unencrypted paths only
- **Chicken-and-egg protection** — encryption unlock instructions are force-classified as Essential
- **CI dependency scanning** — detects scripts that regex-scan instruction files; keeps matched content inline
- **Zero-loss validation** — verifies content after optimization; aborts if anything is missing
- **Format-native extraction** — uses each format's own sub-doc mechanism (imports, nested files, path-scoped instructions) rather than generic markdown links

## When NOT to Use

The skill will decline to optimize if:

- CLAUDE.md is under 200 lines (Anthropic's recommended range)
- AGENTS.md is under 200 lines and combined chain is under 32KB
- copilot-instructions.md is under 150 lines (~1 page)
- More than 80% of content is Essential

## License

MIT

