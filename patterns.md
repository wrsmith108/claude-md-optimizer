# Progressive Disclosure Patterns

This document defines the reusable patterns for transforming verbose CLAUDE.md files into progressive disclosure structures.

---

## 1. Sub-Documentation Table Pattern

An optional reference appendix listing all extracted sub-documents. Place at the **end** of the rewritten file (not the top) if included — it is a convenience index, not a navigation preamble. Prefer inline contextual references (Patterns 2–4) at the point of relevance instead; they convey *why* the reader would follow the link, not just that a file exists.

```markdown
## Reference Documents

| Document | Description |
|----------|-------------|
| [docker-guide.md](path/docker-guide.md) | Container management, troubleshooting |
| [ci-reference.md](path/ci-reference.md) | Branch protection, CI pipeline |
```

### Rules

- Place at the **end** of the file, not the top
- Only include if the file has many sub-docs and an index adds navigational value
- One row per sub-document; description explains the content, not the topic name
- Use relative paths from the main file to the sub-doc location
- Prefer Patterns 2–4 for the actual in-body references — the table is supplementary

---

## 2. Rich Abstract + Link Pattern

For sections where the sub-doc covers a topic the agent may or may not need. A concrete synopsis — actual values, commands, names — lets the agent answer most questions without opening the file. The link is a trapdoor: only follow it when the synopsis is insufficient.

```markdown
## Testing

Tests use **Vitest** (not Jest). Test files co-located with source, `.unit.spec.ts` suffix
(e.g., `stripe-processor.unit.spec.ts`). Minimum **87% line coverage** enforced in CI for
`src/services/` and `src/api/` — falling below this blocks merge. Mock with `vi.mock()`.
For snapshot testing and full mock patterns, see [docs/testing.md](path/testing.md).
```

The synopsis contains the concrete facts (framework name, suffix, threshold) an agent needs for the most common questions. The file is only opened when detail beyond the synopsis is needed.

### When to Use

The section has reference content that is sometimes but not always needed. The synopsis should contain enough concrete specifics (numbers, names, commands) to be self-sufficient for common questions. If the section is needed in every session, keep it fully inline (Pattern 5) instead.

### Sizing the synopsis

Aim for 3–5 sentences containing:
- The technology/framework name(s) — so the agent doesn't need to open the file just to learn the stack
- The key constraint or threshold — the value that determines correctness
- The most common command — so the agent can act without reading the full file
- The scope of the linked file — what additional detail is there, to help the agent decide whether to follow the link

### Evidence

A controlled usage eval compared four strategies across focused and ambiguous tasks (see `evals/` for fixtures):

| Strategy | Focused task reads | Ambiguous task reads |
|---|---|---|
| Thin inline ref ("For X, see Y") | 2 | 5 |
| Rich abstract + link (this pattern) | **1** | **2** |
| Lazy frontmatter only | 2 | 5 |
| Rich abstract + lazy frontmatter | **1** | **2** |

The rich synopsis eliminated all sub-doc reads on focused tasks (the agent answered from the synopsis alone) and reduced ambiguous task reads from 5 to 2. Lazy frontmatter on sub-docs added no further reduction on top of a rich synopsis — the synopsis is where the savings happen.

---

## 2b. Quick-Reference + Link Pattern

For sections where a single command or rule is essential but details are reference-only. Use when the section is narrow enough that one command line is sufficient — not for multi-faceted topics.

```markdown
## Section Name

**Quick start**: `primary command here`

**Full guide**: [guide-name.md](path/guide-name.md)
```

### When to Use

The section has one essential command/rule but 20+ lines of supporting detail. For sections with multiple key facts (framework, threshold, naming convention), prefer Pattern 2 (Rich Abstract + Link) instead.

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

## 10. @Import Chain Pattern *(CLAUDE.md only)*

Use `@path/to/file` syntax to reference external files from CLAUDE.md. Referenced files load at session start — they are functionally equivalent to inline content, but physically separated.

**Before** (inline verbose content):
```markdown
## Docker Development

All commands run inside Docker: `docker exec myapp-dev-1 npm <cmd>`

### Rebuilding the Container
Step 1: ...
Step 2: ...
[40 more lines]
```

**After** (import reference):
```markdown
## Docker Development

All commands run inside Docker: `docker exec myapp-dev-1 npm <cmd>`

@docs/development/docker-guide.md
```

### When to Use

- CLAUDE.md content that is genuinely essential but too long to keep inline
- Content needed in every session (imported files load unconditionally at start)
- When you want to organize instructions across files without losing adherence

### Rules

- The `@` must be at the start of a line
- Both relative and absolute paths are supported; relative paths resolve from the importing file
- Imported files can recursively import other files (max depth: 5 hops)
- Use `@` imports as a first-class alternative to inline content, not as a link-following pattern
- Prefer `@import` over generic markdown links for CLAUDE.md — the file loads automatically, no manual navigation required

---

## 11. .claude/rules/ Migration Pattern *(CLAUDE.md only)*

Move topic-specific rules from CLAUDE.md into `.claude/rules/topic.md` files. Rules without path frontmatter load at launch; rules with `paths:` frontmatter load only when Claude works with matching files.

**Before** (inline TypeScript rules consuming context for every session):
```markdown
## TypeScript Rules
- Prefer interfaces over type aliases for object shapes
- Always use explicit return types on exported functions
- Never use `any`; use `unknown` and narrow with type guards
[20 more lines of TS-specific rules]
```

**After** (scoped rule file):

`.claude/rules/typescript.md`:
```markdown
---
description: "TypeScript strict-mode rules for src/ and test files"
loading_strategy: "lazy"
paths:
  - "src/**/*.ts"
  - "src/**/*.tsx"
  - "tests/**/*.test.ts"
---

# TypeScript Rules
- Prefer interfaces over type aliases for object shapes
[full content verbatim]
```

CLAUDE.md:
```markdown
## TypeScript
TypeScript strict mode. See `.claude/rules/typescript.md` for detailed rules (auto-loaded for .ts/.tsx files).
```

### When to Use

- Rules that only apply to certain file types or directories
- Large rule sets that don't need to be in context for every session
- Teams that want modular, maintainable rule organization

### Rules

- Place files in `.claude/rules/` (discovered recursively)
- Use descriptive filenames: `testing.md`, `api-design.md`, `security.md`
- `paths:` frontmatter uses glob syntax; omit for unconditional loading
- Rules files are concatenated with CLAUDE.md — no link-following required
- Keep a brief mention in CLAUDE.md so users know the rule exists

---

## 12. Path-Scoped Instructions Pattern *(copilot-instructions.md only)*

Extract language, framework, or directory-specific rules from `.github/copilot-instructions.md` into `.github/instructions/NAME.instructions.md` files with `applyTo:` frontmatter. These load automatically for matching files — no user action needed.

**Before** (bloated repo-wide file with language-specific rules):
```markdown
# Copilot Instructions

## Project Overview
...

## TypeScript Standards
- Prefer interfaces over type aliases
- Explicit return types on exported functions
[30 lines of TS rules]

## Python Standards
- Use type hints on all functions
- Follow PEP 8
[25 lines of Python rules]
```

**After**:

`.github/copilot-instructions.md`:
```markdown
# Copilot Instructions

## Project Overview
...

## Language Standards
- TypeScript: see `.github/instructions/typescript.instructions.md`
- Python: see `.github/instructions/python.instructions.md`
```

`.github/instructions/typescript.instructions.md`:
```markdown
---
description: "TypeScript and React coding standards"
loading_strategy: "lazy"
applyTo: "**/*.ts,**/*.tsx"
---

## TypeScript Standards
[full content verbatim]
```

`.github/instructions/python.instructions.md`:
```markdown
---
description: "Python coding standards and type-hint conventions"
loading_strategy: "lazy"
applyTo: "**/*.py"
---

## Python Standards
[full content verbatim]
```

Directory-scoped example (backend API only):
```markdown
---
description: "FastAPI backend conventions for src/api/"
loading_strategy: "lazy"
applyTo: "src/api/**"
---
```

### When to Use

- Copilot instruction files with language-specific or directory-specific sections
- Rules that only need to apply to certain parts of the codebase
- Large monorepos where different teams need different rules

### Rules

- Files must be in `.github/instructions/` (or subdirectories)
- Filename must end with `.instructions.md`
- `applyTo:` (required): glob pattern(s), comma-separated. Use file-type globs (`**/*.ts`) for language rules; use directory globs (`src/api/**`) for directory rules; combine both when appropriate (`src/api/**/*.ts`)
- If additional `paths:` or `globs:` scoping would clarify intent for other tools (e.g., language servers or custom agents), include them in the frontmatter alongside `applyTo:`
- `excludeAgent:` (optional): `"code-review"` or `"cloud-agent"` to restrict to one consumer
- `description:` (required): one-line summary of what this file governs — surfaces in frontmatter readers without opening the file
- `loading_strategy: "lazy"` (required): signals that this file loads on demand, not at session start
- Path-scoped files **supplement** (not replace) the repo-wide file — both apply when a file matches
- Keep a pointer in the repo-wide file so humans can discover the scoped rules

---

## 13. Contextual Reference Pattern *(AGENTS.md and copilot-instructions.md)*

Neither AGENTS.md nor copilot-instructions.md has an `@import` mechanism that auto-loads referenced files. Progressive disclosure still works — but the reader (human or AI) must follow the link manually. To make that worthwhile, the link text must carry its own justification so the agent knows *why* to look, not just *where*.

Use this form consistently:

```
For <reason>, see [<dir>/<ref>.md](<dir>/<ref>.md).
```

### Directory Conventions

| Format | Sub-doc location | Example link |
|--------|-----------------|--------------|
| AGENTS.md | `.agents/instructions/<ref>.md` (repo root) | `For container rebuild steps, see [.agents/instructions/docker.md](.agents/instructions/docker.md).` |
| copilot-instructions.md | `.github/copilot/<ref>.md` | `For container rebuild steps, see [.github/copilot/docker.md](.github/copilot/docker.md).` |

Using a predictable subdirectory (`.agents/instructions/` or `.github/copilot/`) keeps all reference sub-docs in one place and avoids cluttering the repo root.

### Before / After (AGENTS.md example)

**Before** (verbose inline Docker section):

```markdown
## Docker Development

All code runs inside Docker. Start the dev environment:

    docker compose --profile dev up -d

Rebuild after native npm dependency changes:

    docker compose --profile dev down
    docker compose --profile dev build --no-cache
    docker compose --profile dev up -d

DNS failures appear as `getaddrinfo EAI_AGAIN`. Fix:

    docker network prune -f

[40 more lines of troubleshooting steps]
```

**After** (inline essential + reference):

```markdown
## Docker Development

All code runs inside Docker: `docker compose --profile dev up -d`

For rebuild procedures and DNS troubleshooting, see [.agents/instructions/docker.md](.agents/instructions/docker.md).
```

`.agents/instructions/docker.md`:
```markdown
---
description: "Docker rebuild procedures and DNS troubleshooting for the dev environment"
loading_strategy: "lazy"
---

[full extracted content verbatim]
```

### Before / After (copilot-instructions.md example)

**Before** (verbose inline CI section):

```markdown
## CI & Pull Requests

- Run `npm test` and `npm run lint` before opening a PR
- PR titles must follow: `[scope] description`
- Branch protection: requires 2 approvals, green CI, no force-push to main
- CI runs: lint, unit tests, integration tests, Docker build, deploy preview
[25 more lines of CI pipeline detail]
```

**After**:

```markdown
## CI & Pull Requests

- Run `npm test` and `npm run lint` before opening a PR
- PR titles: `[scope] description`

For branch protection rules, required checks, and CI pipeline details, see [.github/copilot/ci.md](.github/copilot/ci.md).
```

`.github/copilot/ci.md`:
```markdown
---
description: "CI pipeline stages, branch protection rules, and required checks"
loading_strategy: "lazy"
---

[full extracted content verbatim]
```

### When to Use

Use Pattern 13 when:
- The section is too long to keep inline and the content is not path-specific (so Pattern 12 doesn't apply)
- The content is reference material (troubleshooting, detailed procedures, large config tables) rather than rules that must always be in context
- You want the agent to be able to consult the detail when it encounters the relevant task

Do **not** use Pattern 13 for Essential content — the agent must follow a link to access it, which creates friction. If the content is needed every session, keep it inline (Pattern 5) or use a format-native auto-loading mechanism.

### Rules

- **Reason before link**: always state why to follow the link before the link itself; a bare "See [.agents/instructions/docker.md]" gives the agent no signal on whether to bother
- **One sentence, one reason**: each reference sentence covers one topic; don't stack multiple sub-docs into one sentence
- **Verbatim extraction**: sub-doc content is copied exactly from the original file, never summarized
- **Relative paths**: use paths relative to the main file's location
- **Frontmatter required**: every sub-doc must have YAML frontmatter with `description:` (one-line summary) and `loading_strategy: "lazy"`. If the content is scoped to specific paths or globs, include those too.
- **Self-contained sub-docs**: each sub-doc should open with a `#` heading so it's readable standalone (the frontmatter `description:` covers the brief summary)

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
What format is the file?
├─ CLAUDE.md → Is the section needed in EVERY session?
│   ├─ YES → Is it too long (>50 lines) to keep inline?
│   │   ├─ YES → Does it have multi-line code blocks?
│   │   │   ├─ YES → Terse Agent Hint (Pattern 8) + @Import Chain (Pattern 10)
│   │   │   └─ NO → @Import Chain Pattern (Pattern 10)
│   │   └─ NO → Does it contain multi-line code blocks (> 3 lines)?
│   │       ├─ YES → Terse Agent Hint Pattern (Pattern 8)
│   │       └─ NO → Inline Essential Pattern (Pattern 5)
│   └─ NO → Is it a rule scoped to specific file types?
│       ├─ YES → .claude/rules/ Migration Pattern (Pattern 11)
│       └─ NO → Continue to generic tree...
│
├─ copilot-instructions.md → Is the section language/framework/directory-specific?
│   ├─ YES → Path-Scoped Instructions Pattern (Pattern 12)
│   └─ NO → Is it reference/procedural content (not needed every session)?
│       ├─ YES → Contextual Reference Pattern (Pattern 13) → .github/copilot/<ref>.md
│       └─ NO → Continue to generic tree...
│
├─ AGENTS.md → Is it subsystem-specific content (belongs to one service/package)?
│   ├─ YES → Nested AGENTS.md in that subdirectory
│   └─ NO → Is it reference/procedural content (not needed every session)?
│       ├─ YES → Contextual Reference Pattern (Pattern 13) → .agents/instructions/<ref>.md
│       └─ NO → Continue to generic tree...
│
└─ generic → Continue to generic tree...

Generic tree (applies to all formats after format-specific check):
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
