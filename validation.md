# Validation Methodology

This document describes the validation process to ensure zero information loss during CLAUDE.md optimization.

## 1. Validation Philosophy

The optimization must be **lossless**. Every line of content from the original CLAUDE.md must exist either in the new CLAUDE.md or in a sub-document. Content is extracted **verbatim** — never summarized, paraphrased, or omitted.

**Core Principle**: The optimization is a structural transformation, not a content transformation. We reorganize where information lives, but never alter what it says.

## 2. Pre-Optimization Baseline

Before making any changes, capture these metrics:

```bash
# Count total lines
wc -l CLAUDE.md

# Count sections (## headers)
grep -c "^## " CLAUDE.md

# List all sections with line numbers
grep -n "^## " CLAUDE.md
```

**Store these values for comparison:**
- Original line count
- Original section count
- Section list with line ranges

## 3. Post-Optimization Validation

### Step 1: Line Count Comparison

```bash
# Count new CLAUDE.md
wc -l CLAUDE.md

# Count all sub-documents
wc -l path/to/sub-docs/*.md

# Calculate total
# Total = new CLAUDE.md + all sub-docs
```

**Expected outcome**: Total lines (CLAUDE.md + sub-docs) >= original CLAUDE.md lines

**Why larger?** The new total may be slightly larger due to:
- Sub-document headers and titles added
- Sub-Documentation Table added to CLAUDE.md
- Link lines added in CLAUDE.md for extracted sections
- Markdown formatting for readability

**Pass criteria**: `new_total >= original_lines - 5%`

The 5% tolerance accounts for deduplicated headers and consolidated formatting.

### Step 2: Reduction Verification

Calculate the reduction percentage:

```bash
# Reduction formula
reduction_percent = (new_claude_lines / original_lines) * 100
```

**Pass criteria**: New CLAUDE.md < 50% of original line count

**Why 50%?** If reduction is less than 50%, the optimization provides minimal benefit and may not justify the complexity of sub-documents.

### Step 3: Link Integrity

Verify every link in the new CLAUDE.md resolves to an existing file:

```bash
# Extract all markdown links from CLAUDE.md
grep -oE '\[.*?\]\(.*?\)' CLAUDE.md | grep -oE '\(.*?\)' | tr -d '()'

# For each link, verify the file exists
# Relative links resolve from CLAUDE.md's directory
```

**Verification method:**
1. Extract all `[text](path)` patterns
2. Parse the `path` portion
3. Resolve relative to CLAUDE.md location
4. Check file existence with `[ -f "$path" ]`

**Pass criteria**: Zero broken links

### Step 4: Sub-Documentation Table Verification

Verify the Sub-Documentation Table exists and all entries have valid links:

```bash
# Check table exists
grep -A 20 "## Sub-Documentation" CLAUDE.md | grep -c "\[.*\.md\]"
```

**Verification checklist:**
- [ ] Sub-Documentation Table present in CLAUDE.md
- [ ] Table has at least 1 entry
- [ ] All entries follow format: `| Category | [filename](path) | Description |`
- [ ] All linked files exist
- [ ] All sub-documents are listed (no orphans)

**Pass criteria**: Sub-Documentation Table present with >= 1 entry, all links valid

### Step 5: Essential Content Preservation

Verify critical content remains in CLAUDE.md (not extracted).

**Essential content checklist:**
- [ ] Build/run commands table present
- [ ] Code quality rules present (look for "zero warnings", "zero errors", MUST, NEVER)
- [ ] Test file location patterns present (if they existed in original)
- [ ] Security/secret handling rules present (look for "secret", "API key", "never expose")
- [ ] Encryption unlock instructions present (if encryption detected)
- [ ] Package/project structure present
- [ ] Behavioral reminders present at bottom of file

**Validation method:**

```bash
# Check for imperative language patterns
grep -E "(MUST|NEVER|ALWAYS|REQUIRED|IMPORTANT)" CLAUDE.md | wc -l

# Check for command patterns
grep -E "^(npm|docker|git|npx|claude) " CLAUDE.md | wc -l

# Check for behavioral reminders
tail -20 CLAUDE.md | grep -i "reminder"
```

**Pass criteria**: All essential content types that existed in original are present in new CLAUDE.md

### Step 6: Encryption Safety

If encryption was detected in Phase 2 (constraint detection):

- [ ] Sub-documents are in unencrypted directory
- [ ] No encryption unlock instructions were moved to sub-docs
- [ ] The user can unlock the project using only CLAUDE.md
- [ ] Encryption sections remain intact and accessible

**Verification method:**

```bash
# Check if sub-docs directory is in .gitattributes filter rules
git check-attr filter path/to/sub-docs/*.md

# Should return "unspecified" or "unset", not "git-crypt"
```

**Pass criteria**: Sub-documents are NOT encrypted, core encryption instructions remain in CLAUDE.md

## 4. Validation Report Format

Present results to the user after optimization:

```
Optimization Results
====================

CLAUDE.md:     1,212 → 256 lines  (-79%)
Sub-documents: 5 files, 642 lines
Total content: 898 lines           (no loss)

Validation Checks:
  [PASS] Line count: 898 >= 1,212 original (accounting for added headers/links)
  [PASS] Reduction: 256 lines = 21% of original (target: < 50%)
  [PASS] Links: 12 valid, 0 broken
  [PASS] Sub-Documentation Table: present, 5 entries
  [PASS] Essential content: all critical sections preserved
  [PASS] Encryption safety: sub-docs in unencrypted directory

Sub-Documents Created:
  docs/development/docker-guide.md      155 lines
  docs/development/git-crypt-guide.md   122 lines
  docs/development/ci-reference.md      103 lines
  docs/development/deployment-guide.md  136 lines
  docs/development/claude-flow-guide.md 126 lines
```

**Report structure:**
1. Metrics summary (before/after line counts, reduction %)
2. Validation checks (PASS/FAIL for each step)
3. Sub-document manifest (list of created files with sizes)

## 5. Failure Modes

### Total lines decreased

**Symptom**: `new_total < original_lines - 5%`

**Cause**: Content was accidentally omitted during extraction.

**Fix**:
1. Diff the original and new CLAUDE.md to find missing sections
2. Check sub-documents for incomplete extraction
3. Re-extract the missing content verbatim
4. Re-run validation

**Prevention**: Use exact line range extraction (preserve blank lines, indentation)

### Broken links

**Symptom**: Link verification finds non-existent files

**Cause**: Sub-document path incorrect or file not created.

**Fix**:
1. Verify sub-doc directory exists
2. Check all files were written successfully
3. Verify relative path resolution from CLAUDE.md location
4. Update links or re-create missing files

**Prevention**: Test link resolution before writing CLAUDE.md

### Essential content missing

**Symptom**: Imperative rules, commands, or constraints not found in CLAUDE.md

**Cause**: A rule or command table was incorrectly classified as "Reference".

**Fix**:
1. Identify the missing essential section
2. Move the section back to CLAUDE.md from the sub-document
3. Update the Sub-Documentation Table to remove or reclassify
4. Re-run validation

**Prevention**: Use conservative classification — when in doubt, keep it Essential

### Sub-docs in encrypted directory

**Symptom**: `git check-attr filter` shows sub-docs are encrypted

**Cause**: Constraint detection missed an encryption rule or chose wrong directory.

**Fix**:
1. Move sub-docs to the fallback location (`claude-md-docs/` in project root)
2. Update all links in CLAUDE.md to point to new location
3. Verify new location is NOT in `.gitattributes` filter rules
4. Re-run validation

**Prevention**: Test directory encryption status before choosing location

### Reduction insufficient (< 50%)

**Symptom**: New CLAUDE.md is still > 50% of original size

**Cause**: Too many sections classified as "Essential" or not enough content to extract.

**Fix**:
1. Review categorization — consider reclassifying sections
2. Use the Split Pattern for borderline sections (keep summary, extract details)
3. If still insufficient, abort optimization (see "When NOT to Optimize" below)

**Note**: This is acceptable for files under 600 lines; not every CLAUDE.md needs optimization.

## 6. When NOT to Optimize

The skill should decline to optimize if:

- **CLAUDE.md is already under 300 lines** — not worth the overhead
- **More than 80% of content is Essential** — nothing meaningful to extract
- **The project has no documentation directory** — and creating one would be inappropriate
- **The CLAUDE.md is primarily behavioral rules** — minimal reference content

**In these cases**, report the analysis but recommend no action:

```
Analysis Complete
=================

CLAUDE.md: 280 lines
Essential: 240 lines (86%)
Reference: 40 lines (14%)

Recommendation: No optimization needed.
The file is already compact and most content is essential.
```

**Decision criteria:**
- If `essential_percent > 80%`, decline optimization
- If `original_lines < 300`, decline optimization
- If `extractable_lines < 200`, decline optimization (not worth creating sub-docs)

## 7. Rollback Procedure

If validation fails critically (data loss detected), rollback immediately:

```bash
# Restore original CLAUDE.md from backup
cp CLAUDE.md.backup CLAUDE.md

# Remove created sub-documents
rm -rf path/to/sub-docs/

# Notify user
echo "Optimization failed validation. Original restored."
```

**Always create backup before starting optimization.**

## 8. Manual Verification

In addition to automated checks, recommend the user perform manual spot-checks:

1. **Open original and new CLAUDE.md side-by-side** — verify sections match expectations
2. **Test a few links** — click through to sub-documents, confirm content is correct
3. **Search for a known phrase** — use grep to find it in either CLAUDE.md or sub-docs
4. **Read the Sub-Documentation Table** — ensure descriptions are accurate

**User confidence is the final validation step.**

## 9. Validation Logging

Log all validation steps to a temporary file for debugging:

```bash
# Example log format
echo "Pre-optimization line count: 1212" >> validation.log
echo "Post-optimization CLAUDE.md: 256" >> validation.log
echo "Sub-documents total: 642" >> validation.log
echo "[PASS] Line count validation" >> validation.log
echo "[PASS] Link integrity" >> validation.log
```

**Include log path in final report** so user can review if needed.

## 10. Validation Script Template

```bash
#!/bin/bash
# validation.sh - Post-optimization validation

ORIGINAL_LINES=1212  # From pre-optimization baseline
NEW_CLAUDE=$(wc -l < CLAUDE.md)
SUB_DOCS=$(find docs/development -name "*.md" -exec wc -l {} + | tail -1 | awk '{print $1}')
TOTAL=$((NEW_CLAUDE + SUB_DOCS))

echo "Validation Results"
echo "=================="
echo "Original: $ORIGINAL_LINES lines"
echo "New CLAUDE.md: $NEW_CLAUDE lines"
echo "Sub-documents: $SUB_DOCS lines"
echo "Total: $TOTAL lines"

# Check line count
if [ $TOTAL -ge $((ORIGINAL_LINES - ORIGINAL_LINES / 20)) ]; then
  echo "[PASS] Line count"
else
  echo "[FAIL] Line count - data loss detected"
  exit 1
fi

# Check reduction
REDUCTION=$((NEW_CLAUDE * 100 / ORIGINAL_LINES))
if [ $REDUCTION -lt 50 ]; then
  echo "[PASS] Reduction ($REDUCTION%)"
else
  echo "[FAIL] Reduction insufficient ($REDUCTION%)"
  exit 1
fi

# Check links
BROKEN=$(grep -oE '\[.*?\]\(.*?\)' CLAUDE.md | grep -oE '\(.*?\)' | tr -d '()' | while read link; do
  [ -f "$link" ] || echo "$link"
done | wc -l)

if [ $BROKEN -eq 0 ]; then
  echo "[PASS] Links"
else
  echo "[FAIL] $BROKEN broken links"
  exit 1
fi

echo "All checks passed."
```

This script provides automated validation and can be adapted to any project structure.
