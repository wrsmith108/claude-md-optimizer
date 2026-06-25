---
name: "claude-md-optimizer"
version: "2.2.0"
description: "Optimize oversized agent instruction files using progressive disclosure. Handles CLAUDE.md (Claude Code), AGENTS.md (OpenAI Codex / GitHub Copilot agents), and copilot-instructions.md (GitHub Copilot). Auto-detects all three formats in your repo, analyzes content tiers, creates sub-documents using each format's native mechanism, and rewrites the main file with zero information loss. Triggers: optimize CLAUDE.md, AGENTS.md too long, reduce copilot-instructions.md, agent instructions file too large, shrink context file, apply progressive disclosure, instructions file bloated"
tags: ["claude-md", "agents-md", "copilot-instructions", "optimization", "progressive-disclosure", "context-window", "documentation"]
---

# Agent Instructions Optimizer

**Behavioral Classification: Guided Decision (ASK, THEN EXECUTE)**

This skill analyzes CLAUDE.md files and proposes optimizations using progressive disclosure patterns. It NEVER makes destructive changes without explicit user approval. Always present the plan, get confirmation, then execute.

## Quick Start

### Phase 1: Detect & Analyze
1. Scan the repo for agent instruction files:
   - `./CLAUDE.md` or `./.claude/CLAUDE.md` (Claude Code)
   - `./AGENTS.md` and any nested `AGENTS.md` in subdirectories (OpenAI Codex / GitHub Copilot)
   - `.github/copilot-instructions.md` and `.github/instructions/*.instructions.md` (GitHub Copilot)
2. Report each file found with its line count and the format's optimization threshold (see [Supported Formats](#supported-formats))
3. If multiple files need optimization, confirm with the user which to address first
4. Parse the selected file into sections and categorize each:
   - **Essential** - Required for every session (build commands, test patterns, security rules)
   - **Reference** - Needed occasionally (architecture docs, API references, troubleshooting)
   - **Redundant** - Duplicates existing docs or overly verbose

### Phase 2: Detect Constraints
1. Check `.gitattributes` for `filter=git-crypt diff=git-crypt` patterns
2. Identify encrypted paths (docs/, .claude/, etc.)
3. Find safe unencrypted directory for sub-documents (prefer project root or subdirs)
4. **Scan for CI machine-readable dependencies** (see [constraints.md](./constraints.md))
5. **Identify the format's native sub-doc mechanism** (see [Supported Formats](#supported-formats) and [constraints.md](./constraints.md))

### Phase 3: Plan
Present categorization table to user:
```
| Section | Lines | Category | Target | Rationale |
|---------|-------|----------|--------|-----------|
| Docker Commands | 45 | Reference | docker-guide.md | Used <10% of sessions |
| CI Health | 120 | Reference | ci-reference.md | Detailed reference rarely needed |
| Git-Crypt Setup | 30 | Essential | Keep inline | Unlock instructions (chicken-and-egg) |
```

For **CLAUDE.md**: also show whether sections can use `@import` or move to `.claude/rules/`.
For **copilot-instructions.md**: show whether sections are candidates for path-scoped `.instructions.md` files.

Wait for user approval before proceeding.

### Phase 4: Review
User confirms:
- Categorization is correct
- Sub-document names make sense
- Essential content remains inline
- Safe directory for sub-docs identified

### Phase 5: Execute
1. Create sub-documents in safe directory (verbatim extraction, NO summarization)
2. For **CLAUDE.md**: use `@path/to/file` imports or `.claude/rules/*.md` as the extraction target when possible — these are native to the format and load at session start
3. For **AGENTS.md**: use nested `AGENTS.md` files in subdirectories for subsystem-specific content; use `.agents/instructions/<ref>.md` + "For `<reason>`, see [.agents/instructions/\<ref>.md]" links for general reference content
4. For **copilot-instructions.md**: extract language/framework rules to `.github/instructions/NAME.instructions.md` with `applyTo:` frontmatter; use `.github/copilot/<ref>.md` + "For `<reason>`, see [.github/copilot/\<ref>.md]" links for reference content that isn't path-specific
5. Rewrite the main file: replace each extracted section with a **rich inline synopsis** followed by a link (Pattern 2). The synopsis must contain enough concrete specifics (framework names, thresholds, ports, naming conventions) that an agent can answer common questions without opening the sub-doc. The link is a trapdoor for when more detail is needed.
6. Optionally add a reference table at the **end** of the file if there are many sub-docs and an index adds navigational value (see Pattern 1)
7. Preserve all Essential content inline

### Phase 6: Validate
1. Count lines before/after, report reduction vs. format target (see [Supported Formats](#supported-formats))
2. Diff all extracted content to verify zero information loss
3. Confirm encryption constraints respected
4. Present validation report to user

## Supported Formats

| Format | Primary Location | Optimize If | Native Sub-Doc Mechanism |
|--------|-----------------|-------------|--------------------------|
| CLAUDE.md | `./CLAUDE.md` or `./.claude/CLAUDE.md` | >200 lines | `@path/to/file` imports; `.claude/rules/*.md` |
| AGENTS.md | `./AGENTS.md` (hierarchical) | >200 lines | Nested `AGENTS.md` in subdirectories; `.agents/instructions/<ref>.md` with contextual links |
| copilot-instructions.md | `.github/copilot-instructions.md` | >150 lines | `.github/instructions/NAME.instructions.md`; `.github/copilot/<ref>.md` with contextual links |

### CLAUDE.md
Anthropic recommends **<200 lines** — above this, instruction adherence degrades. Two native extraction paths avoid the overhead of linked sub-docs:
- **`@path/to/file` imports**: Add `@docs/development/docker-guide.md` anywhere in CLAUDE.md; the file loads into context at session start, equivalent to inline content
- **`.claude/rules/*.md`**: Topic-specific rule files with optional `paths:` frontmatter to scope rules to file types (only loads when Claude works with matching files, saving context)

### AGENTS.md
Plain Markdown, no required schema. Codex and GitHub Copilot load files hierarchically (global → repo root → subdirectory → CWD); files closer to CWD take precedence. Codex truncates the combined chain at a configurable budget (`project_doc_max_bytes`, default 32 KiB), so keep the chain within that budget. In Codex, `AGENTS.override.md` at any level takes precedence over `AGENTS.md` at the same level (Codex-specific, not part of the cross-tool agents.md spec).

Two progressive disclosure strategies:
- **Nested `AGENTS.md` files**: move subsystem-specific rules into `services/payments/AGENTS.md` etc. — auto-discovered when working in that directory
- **`.agents/instructions/<ref>.md` + contextual link**: for reference content that doesn't belong to a specific subsystem, extract to `.agents/instructions/<ref>.md` and link with "For `<reason>`, see [.agents/instructions/\<ref>.md](.agents/instructions/\<ref>.md)." — the link text carries intent so the agent knows when to follow it. Each sub-doc should include a `description:` frontmatter field (a one-line summary).

### copilot-instructions.md
Repository-wide instructions in `.github/copilot-instructions.md` (no frontmatter required). Two progressive disclosure strategies:
- **Path-scoped `.instructions.md` files**: extract language/framework rules to `.github/instructions/NAME.instructions.md` with `applyTo:` frontmatter — load automatically for matching files, no link-following. `applyTo:` is required; also include a `description:` field. If the rules are also scoped to specific directories (not just file types), add directory globs to `applyTo:` or include a `paths:` field alongside it.
- **`.github/copilot/<ref>.md` + contextual link**: for reference content that isn't path-specific, extract to `.github/copilot/<ref>.md` and link with "For `<reason>`, see [.github/copilot/\<ref>.md](.github/copilot/\<ref>.md)."

Path-scoped files **supplement** the repo-wide file — both apply when a file matches.

## Sub-Documentation

| Document | Description |
|----------|-------------|
| [analysis.md](./analysis.md) | Content categorization methodology, format thresholds, and format-specific extraction strategies |
| [patterns.md](./patterns.md) | Progressive disclosure patterns including @import, .claude/rules/, and path-scoped instructions |
| [constraints.md](./constraints.md) | Encryption detection, safe path selection, and format-specific placement constraints |
| [validation.md](./validation.md) | Before/after diffing, per-format pass criteria, and information loss verification |

## Essential Rules

### Safety First
- **ALWAYS** present the plan before making changes
- **NEVER** make destructive edits without explicit user approval
- **ALWAYS** validate zero information loss after optimization

### Format Awareness
- **ALWAYS** identify the file format before choosing an extraction strategy
- **PREFER** the format's native sub-doc mechanism (imports, nested files, path-scoped instructions) over generic markdown links
- **NEVER** move content that is consulted in >50% of sessions

### Content Preservation
- **NEVER** summarize or paraphrase content — verbatim extraction only
- **ALWAYS** keep encryption unlock instructions in main file (chicken-and-egg problem)
- **ALWAYS** keep security rules, test file patterns, behavioral reminders inline
- **NEVER** move content that is consulted in >50% of sessions
- **ALWAYS** keep content that CI scripts parse via regex (see CI Machine-Readable Content below)

### Encryption Safety
- Sub-documents **MUST** be in unencrypted directories
- Check `.gitattributes` for `filter=git-crypt` patterns
- If entire project is encrypted, warn user and recommend decrypted subdirectory

### CI Machine-Readable Content
- CI audit scripts may use regex to scan CLAUDE.md for specific literal strings
- **ALWAYS** scan for scripts that `readFileSync('CLAUDE.md')` or `grep` CLAUDE.md
- Common patterns: deploy commands, function lists, config references
- Content matched by CI regex MUST remain in CLAUDE.md — extraction breaks CI
- See [constraints.md](./constraints.md) for detection methodology

### Progressive Disclosure
- Replace extracted sections with a **rich inline synopsis + link** (Pattern 2): 3–5 sentences with concrete specifics (names, thresholds, commands) followed by the link
- The synopsis is the primary value — most agent questions should be answerable from it without opening the sub-doc
- The link is a trapdoor for detail: include what extra information the sub-doc contains so the agent can judge whether to follow it
- Avoid a table of all sub-docs at the top; an upfront index doesn't convey when each file is needed
- Target: >50% line reduction with zero information loss

## Quick Reference

### Content Tiers
| Tier | Criteria | Action |
|------|----------|--------|
| Essential | Used every session, unlocks workflow, defines behavior | Keep inline |
| Reference | Used occasionally, detailed specs, troubleshooting | Extract to sub-doc |
| Redundant | Duplicates existing docs, overly verbose, historical | Extract or remove |

### Sub-Document Naming
- Use kebab-case domain names: `docker-guide.md`, `ci-reference.md`, `api-docs.md`
- Group related content: `troubleshooting/native-modules.md`
- Avoid generic names: `reference.md`, `docs.md`

### Progressive Disclosure Patterns
1. **Sub-Documentation Table** - Optional reference index (place at *end*, not top)
2. **Rich Abstract+Link** - 3–5 sentence synopsis with concrete specifics + link; agent answers most questions from synopsis alone
3. **Compressed Table+Link** - Condensed table with full version in sub-doc
4. **Summary+Link** - Brief paragraph + link to comprehensive guide

### Target Metrics
| Format | Goal | Information Loss |
|--------|------|-----------------|
| CLAUDE.md | <200 lines (Anthropic guideline) | 0% (content moved, not deleted) |
| AGENTS.md | Meaningful reduction; stay within Codex's `project_doc_max_bytes` budget (32 KiB default) | 0% |
| copilot-instructions.md | ~100 lines (~2 pages) | 0% |

- Essential content remains accessible (inline or via format-native import/reference)
- Session impact: minimal friction for common tasks

## Example Output

**CLAUDE.md**: 850 lines → 175 lines (79% reduction)
- Build commands, security rules, test patterns: kept inline
- Docker guide: extracted to `docs/development/docker-guide.md`, imported with `@docs/development/docker-guide.md`
- CI rules: moved to `.claude/rules/ci.md` (path-scoped to `**/.github/**`)
- Architecture links: replaced with single index link

**AGENTS.md**: 420 lines → 110 lines root + nested subdirectory files (74% reduction)
- Root: core conventions, build commands, PR rules
- `services/payments/AGENTS.md`: payments-specific rules (65 lines)
- `services/auth/AGENTS.md`: auth-specific rules (45 lines)

**copilot-instructions.md**: 300 lines → 85 lines root + path-scoped files (72% reduction)
- Root: project overview, build commands, PR conventions
- `.github/instructions/typescript.instructions.md` (applyTo: `**/*.ts,**/*.tsx`)
- `.github/instructions/python.instructions.md` (applyTo: `**/*.py`)

## Validation Checklist

- [ ] All extracted content is byte-for-byte identical (verified via diff)
- [ ] Sub-documents are in unencrypted directory (or use format's native mechanism)
- [ ] Essential content remains accessible (inline or imported/linked)
- [ ] Extracted sections replaced with inline contextual references ("For `<reason>`, see [file.md]") at the point of relevance
- [ ] All relative links and `@import` paths work
- [ ] User approved the plan before execution
- [ ] Line count is within format target (see Supported Formats)
- [ ] No chicken-and-egg problems (encryption, authentication)
- [ ] CI scripts that regex-scan the instruction file still pass (see CI Machine-Readable Content)

## Error Handling

**Entire project is encrypted**: Warn user, suggest creating `docs/claude/` or similar unencrypted subdirectory.

**User rejects plan**: Ask for feedback, iterate on categorization.

**Information loss detected**: Abort optimization, present diff to user, do not commit changes.

**Ambiguous categorization**: Default to Essential (keep inline), ask user to review.

**File already at target size**: Report analysis but decline to optimize. Thresholds: CLAUDE.md <200 lines, AGENTS.md <200 lines, copilot-instructions.md <150 lines.

## Notes

- This skill is project-agnostic and works with any of the three supported instruction file formats
- Always read the full file before categorizing (context matters)
- Encryption constraints are project-specific, always check `.gitattributes`
- Validation is mandatory, never skip Phase 6
- For CLAUDE.md: prefer `@import` and `.claude/rules/` over generic markdown sub-docs — they're the format's native mechanism
- For AGENTS.md: nested AGENTS.md files in subdirectories keep subsystem context close to the code
- For copilot-instructions.md: path-scoped `.instructions.md` files reduce noise by only loading rules relevant to the file being edited
