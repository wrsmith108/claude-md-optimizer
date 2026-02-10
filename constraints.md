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
# supabase/**
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
  Encrypted:  docs/**, .claude/**, supabase/**
  Exceptions: docs/development/*.md, docs/templates/*.md

Safe sub-doc location: docs/development/ (flat files only)

Chicken-and-egg sections (force Essential):
  - "Git-Crypt (Encrypted Documentation)" (lines 197-312)
  - Contains: git-crypt unlock command, encrypted paths table
```

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
9. **Generate constraint report** for user approval
10. **Document chosen path** in optimization report

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
