# Content Categorization Methodology

This document defines how the claude-md-optimizer analyzes and categorizes CLAUDE.md content for optimization.

## 1. Section Parsing

Parse CLAUDE.md by `##` headers into discrete sections. For each section, measure:

| Metric | Description |
|--------|-------------|
| Line count | Total lines in section (including whitespace) |
| Code block count | Number of fenced code blocks (```...```) |
| Table count | Number of markdown tables |
| Command ratio | Ratio of command examples to prose paragraphs |

These metrics inform tier classification decisions.

## 2. Content Tiers

Three tiers determine the action taken for each section:

| Tier | Criteria | Action |
|------|----------|--------|
| Essential | Rules, constraints, behavioral instructions, quick-reference commands | Keep inline |
| Reference | Detailed guides, step-by-step procedures, verbose examples | Extract to sub-doc |
| Redundant | Content duplicating existing docs (ADRs, READMEs) | Replace with link |

## 3. Essential Heuristics (Keep Inline)

Keep content inline in CLAUDE.md if it matches these patterns:

- **Behavioral rules**: Contains MUST/NEVER/ALWAYS language (e.g., "ALWAYS run in Docker")
- **Compact command tables**: Quick reference tables < 15 lines
- **Error prevention**: Constraints that prevent common mistakes (e.g., test file location patterns)
- **Security rules**: Secret handling, auth patterns, credential policies
- **Setup prerequisites**: Unlock/setup instructions needed before accessing other docs
- **Key identifiers**: Container names, package names, project structure used in every command
- **Critical paths**: File/directory locations referenced by multiple tools
- **License/pricing summary**: One-table overview of tiers and pricing
- **Terse format rule**: Essential multi-line code blocks (> 3 lines) should be compressed to single-line command references with sub-doc links. This reduces both system prompt tokens and execution output tokens when the agent runs the commands.

**Example: Keep Inline**

```markdown
## Build Commands

| Command | Docker (REQUIRED) |
|---------|-------------------|
| Build | `docker exec myapp-dev-1 npm run build` |
| Test | `docker exec myapp-dev-1 npm test` |
| Lint | `docker exec myapp-dev-1 npm run lint` |
```

This is compact (4 lines), contains essential commands, and prevents errors (reminds user of Docker requirement).

## 4. Reference Heuristics (Extract)

Extract content to sub-documents if it matches these patterns:

- **Multi-step procedures**: > 5 sequential steps
- **Detailed troubleshooting**: Symptoms/Root Cause/Fix structure with code examples
- **Large code blocks**: > 10 lines of example code
- **Large tables**: > 10 rows
- **Deep-dive explanations**: "How It Works", architecture details, internal mechanisms
- **Deployment listings**: Full command listings with flags/options
- **Configuration examples**: Multi-option configuration with env vars, file examples
- **Workflow examples**: Agent spawn patterns, orchestration examples
- **Manual workarounds**: Alternate procedures for edge cases
- **Link directories**: Tables that primarily list architecture doc links

**Example: Extract to Sub-Doc**

```markdown
## Troubleshooting Docker DNS Failure

**Symptoms**: `getaddrinfo EAI_AGAIN registry.npmjs.org`, network timeouts

**Root Cause**: Stale Docker bridge networks degrade DNS proxy

**Diagnosis**:
```bash
# Check network count
docker network ls | wc -l
# Test DNS
docker exec container node -e "require('dns').resolve(...)"
```

**Fix**:
```bash
# 1. Restart Docker Desktop
# 2. Prune networks
docker network prune -f
# 3. Restart container
...
```

This is > 20 lines, contains detailed diagnosis steps, and is rarely needed. Extract to `troubleshooting.md`.

## 5. Redundant Heuristics (Replace with Link)

Replace with link if:

- **Minimal added context**: Section says "See [doc](path)" with < 5 lines of unique content
- **Verbatim duplication**: Content duplicates an ADR or architecture doc word-for-word
- **README repetition**: Content already in package README or project root README

**Example: Replace with Link**

```markdown
## Architecture Documentation

For system design details, see:
- [System Overview](docs/architecture/system-overview.md)
- [Skill Dependencies](docs/architecture/skill-dependencies.md)
- [Database Standards](docs/architecture/standards-database.md)
```

Replace entire section with: `**Architecture**: See [docs/architecture/index.md](docs/architecture/index.md)`

## 6. Domain Grouping

Group extracted sections by domain affinity to create coherent sub-documents:

| Domain | Typical Sections | File Name |
|--------|------------------|-----------|
| Docker | Container management, rebuild, native modules, volume cleanup | `docker-guide.md` |
| CI/CD | Pipeline structure, branch protection, change classification, required checks | `ci-reference.md` |
| Deployment | Deploy commands, infrastructure, monitoring, scheduled jobs | `deployment-guide.md` |
| Security | Auth, secrets, encryption setup, git-crypt, API keys | `security-guide.md` |
| Tooling | Dev tools, MCP servers, IDE config, skills, SPARC modes | `tooling-guide.md` |
| Troubleshooting | All troubleshooting sections consolidated by domain | `troubleshooting.md` |
| Testing | Test patterns, benchmarks, integration tests, mocking | `testing-guide.md` |

Aim for 3-5 sub-documents maximum. Over-splitting creates fragmentation.

## 7. Judgment Required Cases

Some content straddles tiers. Document the decision criteria:

| Content Type | Keep Inline If... | Extract If... |
|-------------|-------------------|---------------|
| CI change tiers | Single summary sentence | Full path pattern tables with 20+ rows |
| Docker rebuild guide | 2-line quick reference ("use full rebuild for native modules") | Full scenario decision table with diagnosis steps |
| Edge function table | Compact function name + auth flag reference | Full deploy command listings with explanations |
| Git workflow | Essential commands (3-4 one-liners) | Full rebase/worktree procedures with 10+ steps |
| Tool configuration | Setup command + key env var | Multi-option configuration examples with YAML/JSON |
| Architecture links | Single line with index link | Table with 10+ doc links and descriptions |
| Multi-line bash procedures | Terse hint naming commands (< 3 lines) | Full step-by-step with fenced code blocks (>= 3 lines) |

**Split Strategy**: If a section has both essential summary and verbose details, keep the summary inline and extract the details.

**Example: Split Section**

Keep inline (5 lines):

```markdown
## Docker Development

All commands run inside Docker: `docker exec myapp-dev-1 npm <cmd>`

See [docker-guide.md](docker-guide.md) for container management and troubleshooting.
```

Extract to `docker-guide.md` (80 lines):

- Full rebuild procedures
- Native module troubleshooting
- Volume management
- DNS failure diagnosis

## 8. Output Format

Before making changes, present a categorization table to the user:

```markdown
| Section | Lines | Code Blocks | Tables | Tier | Action | Destination |
|---------|-------|-------------|--------|------|--------|-------------|
| Docker Development | 85 | 6 | 2 | Reference | Extract | docker-guide.md |
| CI Requirements | 120 | 4 | 3 | Split | 25 keep, 95 extract | ci-reference.md |
| Build Commands | 15 | 0 | 1 | Essential | Keep | CLAUDE.md |
| Troubleshooting DNS | 45 | 4 | 0 | Reference | Extract | troubleshooting.md |
| Architecture Docs | 12 | 0 | 1 | Redundant | Replace with link | N/A |
| Git-Crypt Setup | 30 | 3 | 0 | Essential | Keep (security prerequisite) | CLAUDE.md |
| MCP Server Config | 65 | 5 | 2 | Reference | Extract | tooling-guide.md |
```

**Action Key**:

- **Keep**: No changes, stays in CLAUDE.md
- **Extract**: Move to sub-document, leave summary + link in CLAUDE.md
- **Split**: Keep essential summary, extract details
- **Replace with link**: Remove section, add one-line reference

## 9. Analysis Report Structure

The categorization output should include:

### Section Summary

- Total sections analyzed
- Distribution by tier (Essential / Reference / Redundant)
- Total line count before/after optimization

### Category Breakdown

For each section:

- Section name
- Metrics (lines, code blocks, tables)
- Tier assignment
- Rationale (which heuristics triggered the classification)
- Recommended action
- Destination sub-document (if applicable)

### Proposed Structure

Show the optimized CLAUDE.md structure:

```markdown
# CLAUDE.md (Optimized)

## Essential Sections (kept inline)
- Docker Development (5 lines)
- Build Commands (15 lines)
- Git-Crypt Setup (30 lines)

## Reference Sub-Documents
- docker-guide.md (docker-development details)
- ci-reference.md (CI pipeline, branch protection)
- troubleshooting.md (all troubleshooting consolidated)

## Redundant Sections (replaced with links)
- Architecture Documentation → [docs/architecture/index.md]
```

### User Decision Points

Highlight ambiguous cases requiring user input:

```markdown
## Requires Judgment

**Section: CI Change Classification**
- Lines: 45, Tables: 2
- Heuristics: Compact table (Keep) vs detailed path patterns (Extract)
- Recommendation: Split (keep summary table, extract path details)
- User choice: [keep inline] [split] [extract fully]
```

## 10. Edge Cases

### Case 1: Recently Updated Content

If a section has a timestamp or "Updated 2025-XX-XX" marker, bias toward keeping inline (likely reflects recent learnings).

### Case 2: High Command Density

If section is > 50% code blocks with minimal prose, consider keeping inline (quick reference value).

### Case 3: Cross-Linked Sections

If a section is referenced by 3+ other sections, bias toward keeping inline or creating a dedicated sub-doc (high centrality).

### Case 4: Security-Critical Content

Always keep security rules, auth patterns, and secret handling inline (error prevention priority).

### Case 5: One-Time Setup

If content is only needed once (e.g., MCP server install), extract to sub-doc (reduces noise for regular users).

## 11. Validation Checks

After categorization, validate:

- **Essential tier**: No section > 50 lines (if larger, split or extract)
- **Sub-document count**: No more than 5 files (avoid fragmentation)
- **Link hygiene**: All extracted content has a summary + link in CLAUDE.md
- **Domain coherence**: Each sub-doc has 3+ related sections (avoid single-section files)

## 12. Metrics

Report optimization metrics:

| Metric | Before | After |
|--------|--------|-------|
| Total lines | 1200 | 350 |
| Sections | 25 | 12 |
| Code blocks | 40 | 8 |
| Sub-documents created | 0 | 4 |
| Estimated read time | 15 min | 4 min |

---

This methodology ensures systematic, consistent categorization while preserving critical context and providing clear user decision points.
