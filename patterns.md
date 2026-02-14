# Progressive Disclosure Patterns

This document defines the reusable patterns for transforming verbose CLAUDE.md files into progressive disclosure structures.

---

## 1. Sub-Documentation Table Pattern

Always placed immediately after the main title in the rewritten CLAUDE.md. This is the navigation index.

```markdown
# Project Configuration

## Sub-Documentation

| Document | Description |
|----------|-------------|
| [docker-guide.md](path/docker-guide.md) | Container management, troubleshooting |
| [ci-reference.md](path/ci-reference.md) | Branch protection, CI pipeline |
```

### Rules

- Always the FIRST section after the title
- One row per sub-document
- Description should be a concise comma-separated list of topics
- Use relative paths from CLAUDE.md to the sub-doc location

---

## 2. Quick-Reference + Link Pattern

For sections where a single command or rule is essential but details are reference-only.

```markdown
## Section Name

**Quick start**: `primary command here`

**Full guide**: [guide-name.md](path/guide-name.md)
```

### When to Use

The section has one essential command/rule but 20+ lines of supporting detail.

---

## 3. Compressed Table + Link Pattern

For troubleshooting sections where a lookup table provides immediate value but diagnostic steps are reference.

```markdown
## Troubleshooting

| Problem | Fix |
|---------|-----|
| Container won't start | `docker compose down && docker volume rm ... && docker compose up -d` |
| DNS failure | `docker network prune -f` then restart |
| Native module errors | `npm rebuild <native-module>` |

**Detailed diagnostics**: [troubleshooting.md](path/troubleshooting.md)
```

### When to Use

Multiple troubleshooting scenarios exist, each with Symptoms/Root Cause/Fix sections that are verbose.

---

## 4. Summary + Link Pattern

For feature/tool sections where a one-liner summary is enough for most sessions.

```markdown
## Feature Name

Feature does X. Supports modes A, B, C. Configure via `config-file`.

**Full guide**: [feature-guide.md](path/feature-guide.md)
```

### When to Use

The feature is well-understood and only needs reference when actively configuring.

---

## 5. Inline Essential Pattern

For sections that MUST remain fully inline. No extraction, no linking.

```markdown
## Build Commands

| Command | Docker |
|---------|--------|
| Build | `docker exec dev-1 npm run build` |
| Test | `docker exec dev-1 npm test` |
| Lint | `docker exec dev-1 npm run lint` |
```

### When to Use

Content is needed in every single session. Rules, commands, constraints.

---

## 6. Split Pattern

For sections where part is essential and part is reference. Keep the essential portion inline, extract the rest.

```markdown
## CI Requirements

| Category | Requirement |
|----------|-------------|
| ESLint | Zero warnings, zero errors |
| TypeScript | Strict mode |
| Tests | 100% pass rate |

**When CI fails**: Don't merge. Check logs. Run `preflight` locally.

**Change tiers, branch protection, CI scripts**: [ci-reference.md](path/ci-reference.md)
```

### When to Use

A section has a compact rule table (essential) followed by detailed operational procedures (reference).

---

## 7. Sub-Document Structure

Each extracted sub-document should follow this structure:

```markdown
# Topic Name

Brief description of what this guide covers.

## Section 1

[Verbatim content extracted from CLAUDE.md]

## Section 2

[Verbatim content extracted from CLAUDE.md]
```

### Rules

- Content is extracted VERBATIM — no summarization or paraphrasing
- Each sub-doc is self-contained with its own heading hierarchy
- Use relative links back to CLAUDE.md or other sub-docs where relevant
- Sub-docs use `#` for their title (they're standalone documents)

---

## 8. Terse Agent Hint Pattern

For Essential content that stays inline but contains multi-line code blocks. Instead of a full bash script, compress to a single-line command reference with a sub-doc link for the full procedure.

**Before** (high context cost — loads every turn AND generates verbose terminal output):

```markdown
## Git-Crypt

**Rebase workaround** (smudge filter):
```bash
git format-patch -N HEAD -o /tmp/patches/
git fetch origin main && git reset --hard origin/main
git am /tmp/patches/*.patch
```
```

**After** (low context cost — one line in system prompt, no executable block):

```markdown
## Git-Crypt

- **Rebase fails** (smudge filter): save with `git format-patch`, reset to `origin/main`, re-apply with `git am` — see [git-crypt-guide.md](path/git-crypt-guide.md)
```

### When to Use

- Essential content that must stay inline (needed every session)
- Contains multi-line code blocks (> 3 lines)
- The code block would generate verbose terminal output if executed by the agent
- The procedure is a sequence of commands that can be described in one sentence

### Rules

- Name the key commands inline (e.g., `git format-patch`, `git am`) so the agent knows what tools to use
- Link to the sub-doc for the full step-by-step procedure
- Keep the hint to a single bullet point or sentence
- Do not include fenced code blocks — use inline code spans only

---

## 9. Naming Conventions

Sub-documents use kebab-case with domain-based names.

### Common Domain Names

| Domain | Description |
|--------|-------------|
| `docker-guide.md` | Docker container management |
| `ci-reference.md` | CI/CD pipeline configuration |
| `deployment-guide.md` | Deployment procedures and workflows |
| `security-guide.md` | Security practices and authentication |
| `tooling-guide.md` | Development tools and integrations |
| `troubleshooting.md` | Diagnostic procedures and fixes |
| `git-crypt-guide.md` | Git-crypt setup and workflows |
| `claude-flow-guide.md` | Claude-Flow orchestration and agents |
| `testing-guide.md` | Test strategy and execution |
| `architecture-reference.md` | Architecture documentation index |

### Naming Rules

- Names should reflect the domain, not the CLAUDE.md section header
- Use descriptive suffixes: `-guide` (procedures), `-reference` (lookups), `-workflow` (processes)
- Keep names under 30 characters
- Avoid project-specific terminology

---

## Pattern Selection Decision Tree

Use this decision tree to select the appropriate pattern:

```
Is this section needed in EVERY session?
├─ YES → Does it contain multi-line code blocks (> 3 lines)?
│   ├─ YES → Terse Agent Hint Pattern (Pattern 8)
│   └─ NO → Inline Essential Pattern (Pattern 5)
└─ NO → Continue...
    │
    Does the section have ONE critical command/rule + verbose details?
    ├─ YES → Quick-Reference + Link Pattern (Pattern 2)
    └─ NO → Continue...
        │
        Is this a troubleshooting section with multiple scenarios?
        ├─ YES → Compressed Table + Link Pattern (Pattern 3)
        └─ NO → Continue...
            │
            Can this feature be described in 1-2 sentences?
            ├─ YES → Summary + Link Pattern (Pattern 4)
            └─ NO → Continue...
                │
                Does the section have essential data + reference procedures?
                ├─ YES → Split Pattern (Pattern 6)
                └─ NO → Extract entirely to sub-doc
```

---

## Examples by Section Type

### Commands Section

**Before**: 50 lines of commands with explanations, options, and examples.

**After**: Use Pattern 5 (Inline Essential) — keep the command table, extract explanations to sub-doc.

### Feature Section

**Before**: 30 lines explaining a feature with configuration examples, modes, and edge cases.

**After**: Use Pattern 4 (Summary + Link) — one-liner summary + link to full guide.

### Troubleshooting Section

**Before**: Multiple scenarios with Symptoms/Root Cause/Fix/Prevention.

**After**: Use Pattern 3 (Compressed Table + Link) — quick lookup table + link to diagnostics.

### CI/Build Section

**Before**: Requirements table + 20 lines of branch protection rules + 30 lines of CI scripts.

**After**: Use Pattern 6 (Split) — keep requirements table inline, extract procedures to ci-reference.md.

### Security Section

**Before**: 40 lines of secret management patterns, safe/unsafe examples, AI agent handling.

**After**: Use Pattern 2 (Quick-Reference + Link) — essential rule + link to security-guide.md.

---

## Anti-Patterns

### DO NOT summarize or paraphrase

**Incorrect**:
```markdown
## Docker

Docker is used for consistent environments. See [docker-guide.md](docker-guide.md).
```

**Correct**:
```markdown
## Docker

**All code execution MUST happen in Docker.** Ensures consistent environments, avoids native module issues.

**Quick start**: `docker compose --profile dev up -d`

**Full guide**: [docker-guide.md](docker-guide.md)
```

### DO NOT duplicate content

**Incorrect**:
```markdown
## Docker (in CLAUDE.md)
Container management with `docker compose up -d`.

[docker-guide.md]
# Docker Guide
Container management with `docker compose up -d`.
```

**Correct**: Extract ONCE. Keep only the essential command inline, put details in sub-doc.

### DO NOT break navigation

**Incorrect**:
```markdown
See the Docker guide for more info.
```

**Correct**:
```markdown
**Full guide**: [docker-guide.md](path/docker-guide.md)
```

Always use explicit link syntax with relative paths.

---

## Validation Checklist

After applying patterns, validate the result:

- [ ] Sub-Documentation table is the first section after title
- [ ] Every sub-doc link uses relative paths from CLAUDE.md
- [ ] Essential content (Pattern 5) is kept inline without links
- [ ] No content is duplicated between CLAUDE.md and sub-docs
- [ ] All extracted content is verbatim (no summarization)
- [ ] Sub-docs use `#` for their title (standalone documents)
- [ ] Decision tree logic was followed for pattern selection
- [ ] No anti-patterns present (summarization, duplication, broken links)
