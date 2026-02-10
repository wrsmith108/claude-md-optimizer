# Constraints and Safe Path Selection

## Why Constraints Matter

Sub-documents must be readable without special setup. If a project uses file encryption (git-crypt, SOPS, age), placing sub-docs in encrypted directories defeats the purpose — users cannot read the optimization guide that tells them how to unlock files.

This is the **chicken-and-egg problem**: the instructions for accessing encrypted content must themselves be in unencrypted locations.

## Git-Crypt Detection

### Check for git-crypt

```bash
[ -f .gitattributes ] && grep -q "filter=git-crypt" .gitattributes
```

### Parse encrypted paths

Extract patterns from `.gitattributes`:

```bash
grep "filter=git-crypt" .gitattributes | awk '{print $1}'
# Example output:
# docs/**
# .claude/**
# config/**
```

### Parse exclusions (unencrypted paths within encrypted directories)

```bash
grep "!filter" .gitattributes | awk '{print $1}'
# Example output:
# docs/development/*.md
# docs/templates/*.md
```

### Important: Glob depth matters

- `docs/development/*.md` only matches FLAT files (e.g., `docs/development/guide.md`)
- `docs/development/subfolder/guide.md` would still be encrypted by `docs/**`
- Always place sub-docs as flat files in the excluded directory

## SOPS Detection

### Check for SOPS

```bash
[ -f .sops.yaml ] || [ -f .sops.yml ]
```

### Parse encrypted paths

```bash
grep "path_regex" .sops.yaml
```

SOPS uses regex patterns rather than glob patterns. The principle is the same — identify which paths are encrypted and which are not.

## Age/GPG Detection

### Check for age encryption

```bash
[ -f .age-recipients ] || [ -f .agekey ]
```

### Check for GPG encryption

```bash
ls .gpg-id 2>/dev/null || git config --get filter.gpg.smudge
```

## No Encryption Detected

If no encryption system is found, any directory can be used for sub-documents. Prefer existing documentation directories.

## Safe Path Selection Priority

After detecting encryption constraints, choose the sub-doc directory using this priority order:

| Priority | Location | Condition |
|----------|----------|-----------|
| 1 | `docs/development/` | Exists AND is unencrypted (or no encryption) |
| 2 | `docs/claude-md/` | `docs/` directory is unencrypted |
| 3 | `.claude/reference/` | `.claude/` directory is unencrypted |
| 4 | `claude-md-docs/` | Fallback — create in project root (always unencrypted) |

### Decision Logic

1. **If no encryption**: use `docs/development/` (create if needed)
2. **If encryption detected**:
   - Check if `docs/development/*.md` is explicitly excluded from encryption
   - If yes: use `docs/development/` (flat files only)
   - If no: check if `.claude/` is encrypted
   - If `.claude/` is not encrypted: use `.claude/reference/`
   - Fallback: create `claude-md-docs/` in project root

### Verifying the Chosen Path

After selecting a directory, verify it's actually unencrypted:

```bash
# Create a test file
echo "test" > chosen-dir/test-encryption-check.md

# Check if git-crypt would encrypt it
git-crypt status chosen-dir/test-encryption-check.md
# If output shows "not encrypted", the path is safe

# Clean up
rm chosen-dir/test-encryption-check.md
```

## Chicken-and-Egg Detection

Sections in CLAUDE.md that contain encryption unlock instructions MUST remain in the main file, regardless of their length. These are force-classified as "Essential".

### Detection Patterns

Scan each section for these patterns:

- `git-crypt unlock`
- `git-crypt status`
- `sops --decrypt` or `sops -d`
- `age --decrypt` or `age -d`
- `gpg --decrypt`
- References to encryption key paths (e.g., `GIT_CRYPT_KEY_PATH`, `SOPS_AGE_KEY_FILE`)
- Section headers containing: "Encrypted", "Encryption", "Unlock", "Decrypt", "Git-Crypt", "SOPS"

### What to Keep Inline

When chicken-and-egg is detected, keep in CLAUDE.md:

- The unlock command itself
- The encrypted paths table (so users know what's locked)
- Setup instructions for encryption keys

### What Can Still Be Extracted

Even for encryption-related sections, some content can be extracted:

- Detailed worktree creation procedures
- Manual decrypt workarounds
- Rebase/merge procedures for encrypted repos
- Troubleshooting steps for encryption issues

**Key rule**: The user must be able to unlock the project using only CLAUDE.md, without needing to read any sub-document.

## Constraint Report Format

Present findings to the user before proceeding:

```text
Encryption Constraints:
  System:     git-crypt
  Encrypted:  docs/**, .claude/**, config/**
  Exceptions: docs/development/*.md, docs/templates/*.md

Safe sub-doc location: docs/development/ (flat files only)

Chicken-and-egg sections (force Essential):
  - "Git-Crypt (Encrypted Documentation)" (lines 197-312)
  - Contains: git-crypt unlock command, encrypted paths table
```

## CI Machine-Readable Content Detection

CI audit scripts may use regex to scan CLAUDE.md for specific literal strings. Extracting those strings to sub-documents breaks CI, even though the content still exists in the project.

### Why This Matters

Unlike human readers who can follow a link to a sub-document, CI scripts read CLAUDE.md with `readFileSync` or `grep` and pattern-match against its contents. Moving matched content to a sub-document causes CI to report false negatives — the content exists but the script can't find it.

**Real-world example**: A project's CI audit script scans CLAUDE.md for deploy commands to verify all services are documented. After optimizing CLAUDE.md, the deploy commands were extracted to a sub-document. CI's compliance check failed because the script only reads CLAUDE.md, not sub-documents.

### Detection Methodology

#### Step 1: Find scripts that read CLAUDE.md

```bash
# Search for scripts that read CLAUDE.md directly
grep -rn "CLAUDE.md" scripts/ .github/ --include="*.mjs" --include="*.js" --include="*.ts" --include="*.yml" --include="*.yaml"

# Common patterns to look for:
# readFileSync('CLAUDE.md')
# readFileSync('./CLAUDE.md')
# cat CLAUDE.md
# grep ... CLAUDE.md
```

#### Step 2: Extract regex patterns used against CLAUDE.md

```bash
# Look for regex patterns applied to CLAUDE.md content
grep -A5 "readFileSync.*CLAUDE" scripts/*.mjs scripts/*.js scripts/*.ts 2>/dev/null
grep -A5 "match\|exec\|test\|search" scripts/*.mjs scripts/*.js scripts/*.ts 2>/dev/null | grep -i "claude"
```

#### Step 3: Identify matched content in CLAUDE.md

For each regex pattern found:

1. Run the regex against the current CLAUDE.md
2. Record all matching lines
3. Mark those lines as **CI-Essential** (cannot be extracted)

#### Step 4: Map CI-Essential content to sections

```
CI Machine-Readable Content:
  Script: scripts/audit-standards.mjs
  Pattern: /deploy\s+(\S+)\s+--no-verify/g
  Matches: 12 deploy commands in "Edge Functions" section
  Action: Keep entire deploy commands block in CLAUDE.md
```

### Common CI Scan Patterns

| Pattern Type | Example Regex | What It Matches |
|-------------|---------------|-----------------|
| Deploy commands | `/deploy\s+(\S+)\s+--no-verify-jwt/g` | Function deployment commands |
| Function lists | `/functions\.(name)\b/g` | Function name references |
| Config values | `/"verify_jwt"\s*=\s*false/g` | Configuration settings |
| URL patterns | `/https?:\/\/[^\s]+/g` | URLs and endpoints |
| Version strings | `/version:\s*"([^"]+)"/g` | Version references |

### Classification Rule

Content matched by CI regex patterns is **force-classified as Essential**, regardless of how infrequently humans reference it. The CI script runs on every PR — it's the most frequent consumer of that content.

### Workaround Options

If the matched content is large (>100 lines) and bloats CLAUDE.md:

1. **Update the CI script** to also check sub-documents (preferred, but requires code change)
2. **Keep a minimal stub** in CLAUDE.md that satisfies the regex, with details in sub-doc
3. **Accept the bloat** — CI reliability trumps CLAUDE.md brevity

**Never silently extract CI-scanned content.** Always flag it to the user with the script path, regex pattern, and matched lines.

## Implementation Checklist

When implementing constraint detection:

1. **Scan for encryption systems** (git-crypt, SOPS, age, GPG)
2. **Parse encrypted paths** from configuration files
3. **Identify exceptions** (explicitly unencrypted paths)
4. **Test glob depth** (flat vs. recursive patterns)
5. **Select safe directory** using priority order
6. **Verify chosen path** with test file
7. **Scan for chicken-and-egg content** in sections
8. **Force Essential classification** for unlock instructions
9. **Scan for CI machine-readable dependencies** (scripts that regex-scan CLAUDE.md)
10. **Force Essential classification** for CI-matched content
11. **Generate constraint report** for user approval
12. **Document chosen path** in optimization report

## Edge Cases

### Nested Exclusions

If `.gitattributes` has:
```gitattributes
docs/** filter=git-crypt diff=git-crypt
docs/development/*.md !filter !diff
docs/development/internal/** filter=git-crypt diff=git-crypt
```

Then `docs/development/guide.md` is safe, but `docs/development/internal/secret.md` is encrypted.

### Multiple Encryption Systems

If both git-crypt and SOPS are detected:
- Union the encrypted path sets
- Intersection the exception sets
- Choose the most restrictive constraint

### Partial Chicken-and-Egg

If a section has:
- 20 lines of encryption unlock instructions
- 200 lines of detailed troubleshooting

Split it:
- Keep unlock instructions inline (Essential)
- Extract troubleshooting to sub-doc (Reference)
- Link between them

## Testing Constraints

Before committing sub-docs, verify they're accessible:

```bash
# Simulate fresh clone (no keys)
git clone <repo> /tmp/test-clone
cd /tmp/test-clone

# Try to read sub-docs
cat docs/development/build-commands.md
# Should display plaintext, not binary gibberish

# Clean up
rm -rf /tmp/test-clone
```

If the sub-doc is encrypted, the constraint detection failed. Revisit path selection.
