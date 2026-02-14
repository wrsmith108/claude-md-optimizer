# Changelog

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
