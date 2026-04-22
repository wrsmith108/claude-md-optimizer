# Changelog

## [2.2.0] - 2026-04-22

### Changed

- **Rich abstract + link** is now the primary progressive disclosure pattern (formerly "Quick-Reference + Link"). A 3–5 sentence synopsis with concrete specifics (framework names, thresholds, ports, naming conventions) before the link lets agents answer common questions without opening the sub-doc at all. The link is a trapdoor for when the synopsis is insufficient.
- **Pattern 2 renamed and expanded**: "Quick-Reference + Link" is now "Rich Abstract + Link" with canonical template, sizing guidance, and evidence table. The original single-command form lives on as Pattern 2b.
- Phase 5 step 5 updated: "inline contextual reference" → "rich inline synopsis + link"
- Progressive Disclosure guidance in SKILL.md updated to specify synopsis must contain concrete specifics, not just a reason clause
- Quick Reference Pattern 2 description updated
- README rewritten with a "Progressive Disclosure Research" section documenting measurement methodology, results table, and platform-level lazy loading findings

### Evidence

Usage eval (iteration-3, fixtures in `evals/fixtures/usage-eval-*/`) measured sub-doc file reads across 5 strategies × 2 task types:

| Strategy | Focused task | Ambiguous task |
|---|---|---|
| Thin inline ref | 2 | 5 |
| Lazy frontmatter only | 2 | 5 |
| **Rich abstract + link** | **1** | **2** |
| Rich abstract + lazy frontmatter | 1 | 2 |

Lazy frontmatter (`loading_strategy: lazy` on sub-docs) adds no measurable reduction on top of a rich synopsis. The synopsis is where the savings happen.

## [2.1.0] - 2026-04-22

### Changed

- **Sub-Documentation Table demoted**: Pattern 1 (Sub-Documentation Table) is no longer recommended at the top of rewritten files. An upfront index of all sub-docs doesn't convey *when* each file is relevant. Inline contextual references ("For `<reason>`, see [file.md]") at the point of relevance are now the primary pattern for all three formats.
- Pattern 1 repurposed as an optional end-of-file appendix for repos with many sub-docs
- Phase 5 step 5 updated: rewrite each extracted section with an inline contextual reference instead of producing a table header
- Validation checklist updated: "Sub-Documentation Table present" replaced with "extracted sections have inline contextual references"
- evals.json eval-1 assertion 3 updated to check for inline contextual references instead of a top-of-file table
- Measurement note: a usage eval (iteration-3) compared table-at-top vs. inline-refs across focused and ambiguous tasks. Context consumption (files read) was identical in both cases — the change is justified on structural/cognitive grounds rather than token savings.

## [2.0.0] - 2026-04-21

### Added

- **Multi-format support**: skill now handles CLAUDE.md, AGENTS.md, and copilot-instructions.md
- **Phase 1 auto-detection**: scans repo for all three file types, reports sizes vs. format thresholds
- **Supported Formats section** in SKILL.md: per-format thresholds, native sub-doc mechanisms, format-specific notes
- **Pattern 10: @Import Chain** — CLAUDE.md `@path/to/file` import syntax for loading referenced files at session start
- **Pattern 11: .claude/rules/ Migration** — topic-specific rule files with optional `paths:` path-scoping
- **Pattern 12: Path-Scoped Instructions** — `.github/instructions/NAME.instructions.md` with `applyTo:` frontmatter for GitHub Copilot
- **Section 1b: Format-Specific Thresholds** in analysis.md (CLAUDE.md <200 lines, AGENTS.md <32KB, copilot ~100 lines)
- **Section 13: Format-Specific Extraction Strategies** in analysis.md
- **AGENTS.md Placement Constraints** section in constraints.md (hierarchy, 32KB limit, AGENTS.override.md)
- **copilot-instructions.md Placement Constraints** section in constraints.md (`.github/instructions/`, `applyTo:` frontmatter, `excludeAgent:` field)
- **Pattern 13: Contextual Reference Pattern** — "For `<reason>`, see `.agents/instructions/<ref>.md`" convention for AGENTS.md and "For `<reason>`, see `.github/copilot/<ref>.md`" for copilot-instructions.md; both use dedicated subdirectories (`.agents/instructions/`, `.github/copilot/`) and require the link text to state the reason for following the link
- **Frontmatter convention for all sub-docs**: `description:` (one-line summary) and `loading_strategy: "lazy"` required in frontmatter of every sub-document (`.claude/rules/*.md`, `.github/instructions/*.instructions.md`, `.agents/instructions/<ref>.md`, `.github/copilot/<ref>.md`); globs or `paths:` fields added where content is directory- or file-type-scoped
- Updated decision tree to route AGENTS.md and copilot non-path-specific reference content to Pattern 13
- Updated SKILL.md Supported Formats table and format descriptions to document contextual reference conventions
- Updated analysis.md Section 13 AGENTS.md and copilot-instructions.md extraction strategies and "Choosing Between" table
- Updated constraints.md AGENTS.md and copilot-instructions.md placement rules to document `.agents/instructions/` and `.github/copilot/` directories and required sub-doc frontmatter fields
- Updated validation pass criteria with per-format targets instead of flat 50% reduction rule
- Updated README to document all three supported formats

### Changed

- Skill renamed from `claude-md-optimizer` to `agent-instructions-optimizer`
- Phase 5 (Execute) now specifies format-native extraction strategy per format
- Phase 6 (Validate) references format-specific targets
- "When NOT to Optimize" thresholds updated: CLAUDE.md <200 lines (was 300), copilot <150 lines (new)
- Essential Rules section updated with "Format Awareness" block
- Example Output updated to show all three format scenarios
- README install path updated to `~/.agents/skills/agent-instructions-optimizer`

## [1.1.1] - 2026-02-13

### Added

- **Pattern 8: Terse Agent Hint** — compress multi-line code blocks (> 3 lines) into single-line command references with sub-doc links, reducing both system prompt tokens and execution output tokens
- Updated decision tree with Pattern 8 branch under Inline Essential path
- Terse format heuristic in analysis.md Section 3 (Essential Heuristics)
- Multi-line bash procedures row in analysis.md Section 7 (Judgment Required Cases) table
- Terse Agent Hint row in README pattern table

### Changed

- Naming Conventions renumbered from Pattern 8 to Pattern 9

## [1.1.0] - 2026-01-27

### Added

- CI Machine-Readable Content detection in constraints.md
- Force-classify Essential rule for content matched by CI regex scans
- SKILL.md Phase 2 scanning for CI dependencies

## [1.0.0] - 2026-01-25

### Added

- Initial release with 7 progressive disclosure patterns
- 6-phase guided workflow (Analyze, Detect, Plan, Review, Execute, Validate)
- Encryption-aware sub-document extraction
- Content categorization methodology (Essential, Reference, Redundant tiers)
- Zero information loss validation
- Pattern Selection Decision Tree
