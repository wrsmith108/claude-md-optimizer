---
name: "claude-md-optimizer"
version: "1.1.1"
description: "Optimize oversized CLAUDE.md files using progressive disclosure. Analyzes content tiers, detects encryption constraints, creates sub-documents, and rewrites the main file with a Sub-Documentation Table. Triggers: optimize CLAUDE.md, reduce CLAUDE.md size, CLAUDE.md too long, apply progressive disclosure to CLAUDE.md"
tags: ["claude-md", "optimization", "progressive-disclosure", "context-window", "documentation"]
---

# Claude MD Optimizer

**Behavioral Classification: Guided Decision (ASK, THEN EXECUTE)**

This skill analyzes CLAUDE.md files and proposes optimizations using progressive disclosure patterns. It NEVER makes destructive changes without explicit user approval. Always present the plan, get confirmation, then execute.

## Quick Start

### Phase 1: Analyze
1. Read CLAUDE.md and count total lines
2. Parse into sections and categorize each:
   - **Essential** - Required for every session (build commands, test patterns, security rules)
   - **Reference** - Needed occasionally (architecture docs, API references, troubleshooting)
   - **Redundant** - Duplicates existing docs or overly verbose

### Phase 2: Detect Constraints
1. Check `.gitattributes` for `filter=git-crypt diff=git-crypt` patterns
2. Identify encrypted paths (docs/, .claude/, etc.)
3. Find safe unencrypted directory for sub-documents (prefer project root or subdirs)
4. **Scan for CI machine-readable dependencies** (see [constraints.md](./constraints.md))

### Phase 3: Plan
Present categorization table to user:
```
| Section | Lines | Category | Target | Rationale |
|---------|-------|----------|--------|-----------|
| Docker Commands | 45 | Reference | docker-guide.md | Used <10% of sessions |
| CI Health | 120 | Reference | ci-reference.md | Detailed reference rarely needed |
| Git-Crypt Setup | 30 | Essential | Keep inline | Unlock instructions (chicken-and-egg) |
```

Wait for user approval before proceeding.

### Phase 4: Review
User confirms:
- Categorization is correct
- Sub-document names make sense
- Essential content remains inline
- Safe directory for sub-docs identified

### Phase 5: Execute
1. Create sub-documents in safe directory (verbatim extraction, NO summarization)
2. Rewrite CLAUDE.md with Sub-Documentation Table at top
3. Replace extracted sections with progressive disclosure links
4. Preserve all Essential content inline

### Phase 6: Validate
1. Count lines before/after, report reduction percentage
2. Diff all extracted content to verify zero information loss
3. Confirm encryption constraints respected
4. Present validation report to user

## Sub-Documentation

| Document | Description |
|----------|-------------|
| [analysis.md](./analysis.md) | Content categorization methodology and heuristics |
| [patterns.md](./patterns.md) | Progressive disclosure patterns and templates |
| [constraints.md](./constraints.md) | Encryption detection and safe path selection |
| [validation.md](./validation.md) | Before/after diffing and information loss verification |

## Essential Rules

### Safety First
- **ALWAYS** present the plan before making changes
- **NEVER** make destructive edits without explicit user approval
- **ALWAYS** validate zero information loss after optimization

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
- Use **Sub-Documentation Table** pattern at top of rewritten CLAUDE.md
- Link to sub-docs using relative paths (e.g., `[Docker Guide](./docker-guide.md)`)
- Preserve section headers with brief summary + link
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
1. **Sub-Documentation Table** - Links to extracted domains (use at top of CLAUDE.md)
2. **Quick-Reference+Link** - Inline summary with "See X for details" link
3. **Compressed Table+Link** - Condensed table with full version in sub-doc
4. **Summary+Link** - Brief paragraph + link to comprehensive guide

### Target Metrics
- Line reduction: >50% from original
- Information loss: 0% (verified via diff)
- Essential content: Remains inline
- Session impact: Minimal friction for common tasks

## Example Output

**Before**: 850 lines
**After**: 320 lines (62% reduction)

Extracted:
- `docker-guide.md` (Docker commands and troubleshooting)
- `ci-reference.md` (CI health requirements and branch protection)
- `deployment-guide.md` (Deployment commands and configuration)
- `troubleshooting.md` (Diagnostic procedures consolidated)

Preserved inline:
- Encryption unlock instructions
- Security rules (secret handling, API keys)
- Test file patterns
- Build commands
- Behavioral reminders

## Validation Checklist

- [ ] All extracted content is byte-for-byte identical (verified via diff)
- [ ] Sub-documents are in unencrypted directory
- [ ] Essential content remains inline
- [ ] Sub-Documentation Table is at top of rewritten CLAUDE.md
- [ ] All relative links work
- [ ] User approved the plan before execution
- [ ] Line reduction >50%
- [ ] No chicken-and-egg problems (encryption, authentication)
- [ ] CI scripts that regex-scan CLAUDE.md still pass (see CI Machine-Readable Content)

## Error Handling

**Entire project is encrypted**: Warn user, suggest creating `docs/claude/` or similar unencrypted subdirectory.

**User rejects plan**: Ask for feedback, iterate on categorization.

**Information loss detected**: Abort optimization, present diff to user, do not commit changes.

**Ambiguous categorization**: Default to Essential (keep inline), ask user to review.

## Notes

- This skill is project-agnostic and works with any CLAUDE.md file
- Always read the full CLAUDE.md before categorizing (context matters)
- Encryption constraints are project-specific, always check `.gitattributes`
- Validation is mandatory, never skip Phase 6
