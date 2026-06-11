# Agent: git apply

Apply a unified diff patch file to the working directory. Uses the system `patch` binary.

No `git`, no `python3`. The `patch` command is POSIX standard.

## Steps

### 1. Locate the patch file

Ask the user for the path if not provided. Verify it exists:

```bash
PATCH_FILE="REPLACE_PATH"    # e.g. "changes.patch" or "fix.patch"
[ ! -f "$PATCH_FILE" ] && echo "error: patch file not found: $PATCH_FILE" && exit 1
```

### 2. Inspect the patch (optional preview)

```bash
# Show files affected
grep '^+++ ' "$PATCH_FILE" | sed 's|^+++ b/||;s|^+++ /dev/null|/dev/null|'
```

### 3. Dry run (check if patch applies cleanly)

```bash
patch --dry-run -p1 < "$PATCH_FILE"
```

If the dry run reports errors, report them to the user and stop. Common cause: patch made against a different branch or commit. Tell the user to check out the right base first.

### 4. Apply the patch

```bash
patch -p1 < "$PATCH_FILE"
```

`-p1` strips the leading `a/` or `b/` prefix from diff paths, which is the standard format produced by `diff --label a/<path>` or `git diff`.

### 5. Handle apply errors

If `patch` exits non-zero, it will create `.rej` reject files containing the hunks that failed. Show the user which files have rejects:

```bash
find . -name '*.rej' | while IFS= read -r rej; do
    printf 'CONFLICT: %s\n' "${rej%.rej}"
    cat "$rej"
    printf '---\n'
done
```

The user must manually resolve conflicts in the reject files, then remove the `.rej` files.

### 6. Report

```bash
printf 'Applied: %s\n' "$PATCH_FILE"
# List modified files from patch output
grep '^patching file' /tmp/patch_output 2>/dev/null || true
```

---

## Reverse apply (undo a patch)

To undo a previously applied patch:

```bash
patch -p1 -R < "$PATCH_FILE"
```

`-R` reverses all hunks: additions become deletions and vice versa. Same dry-run workflow applies.

---

## Apply email-format patch (from `git format-patch` or `agents/patch.md` Case 2)

These patches have a `From:` header block before the diff. The `patch` command ignores non-diff lines, so apply the same way:

```bash
patch -p1 < "$PATCH_FILE"
```

---

## Notes on path stripping

| Patch format | strip level |
|---|---|
| `diff -u a/file b/file` (standard) | `-p1` |
| `diff -u file.orig file` (no prefix) | `-p0` |
| Email patches from `git format-patch` | `-p1` |

If paths are wrong, try `-p0` or adjust manually.
