# Agent: git apply

Apply a unified diff patch file to the working directory.

## Step 1: Locate the patch file

Ask the user for the patch file path if not already provided. Read it:

```bash
cat <patch-file>
```

Or use Read tool on the file path.

## Step 2: Validate the patch format

A valid unified diff patch looks like:
```
--- a/path/to/file
+++ b/path/to/file
@@ -1,5 +1,6 @@
 context line
-removed line
+added line
 context line
```

Multi-file patches have multiple `---`/`+++` sections.

## Step 3: Apply the patch

```bash
python3 << 'PYEOF'
import os, re

PATCH_FILE = "REPLACE_PATH"  # replace with actual path

def parse_hunk_header(header_line):
    """Parse '@@ -old_start,old_count +new_start,new_count @@'"""
    m = re.match(r'@@ -(\d+)(?:,(\d+))? \+(\d+)(?:,(\d+))? @@', header_line)
    if not m:
        return None
    old_start = int(m.group(1))
    old_count = int(m.group(2)) if m.group(2) is not None else 1
    new_start = int(m.group(3))
    return old_start, old_count, new_start

def apply_hunk(file_lines, hunk_lines, old_start):
    """Apply one hunk to file_lines (list of strings). old_start is 1-indexed."""
    result = list(file_lines[:old_start - 1])
    pos = old_start - 1

    for raw_line in hunk_lines:
        if not raw_line:
            continue
        ch = raw_line[0]
        text = raw_line[1:]  # content without the +/-/space prefix

        if ch == ' ':   # context — must match
            if pos < len(file_lines):
                result.append(file_lines[pos])
                pos += 1
            else:
                result.append(text)
        elif ch == '+':  # addition
            result.append(text)
        elif ch == '-':  # deletion
            pos += 1     # skip the line from original
        elif ch == '\\': # "No newline at end of file" marker
            if result and result[-1].endswith('\n'):
                result[-1] = result[-1][:-1]

    result.extend(file_lines[pos:])
    return result

with open(PATCH_FILE) as f:
    lines = f.readlines()

i = 0
files_changed = []
errors = []

while i < len(lines):
    # Skip email-style headers (From:, Date:, Subject:)
    if lines[i].startswith(('From ', 'From:', 'Date:', 'Subject:', '---\n', 'diff --git')):
        i += 1
        continue

    if not lines[i].startswith('--- '):
        i += 1
        continue

    # Parse file headers
    old_path = lines[i].strip()[4:]  # strip "--- "
    if old_path.startswith('a/'): old_path = old_path[2:]

    i += 1
    if i >= len(lines) or not lines[i].startswith('+++ '):
        continue
    new_path = lines[i].strip()[4:]  # strip "+++ "
    if new_path.startswith('b/'): new_path = new_path[2:]
    i += 1

    target_path = new_path if new_path != '/dev/null' else old_path
    is_new_file = (old_path == '/dev/null')
    is_deleted = (new_path == '/dev/null')

    # Read current file content
    if os.path.exists(target_path) and not is_new_file:
        with open(target_path) as f:
            content = f.readlines()
    else:
        content = []

    # Apply all hunks for this file
    offset = 0  # track line offset from previous hunks
    while i < len(lines) and lines[i].startswith('@@'):
        parsed = parse_hunk_header(lines[i])
        if not parsed:
            i += 1
            continue
        old_start, old_count, new_start = parsed
        i += 1

        hunk_lines = []
        while i < len(lines) and not lines[i].startswith('@@') \
              and not lines[i].startswith('--- ') \
              and not lines[i].startswith('diff ') \
              and not (lines[i].startswith('From ') and i > 0):
            hunk_lines.append(lines[i].rstrip('\n'))
            i += 1

        # Apply hunk (adjusting old_start by accumulated offset)
        adjusted_start = max(1, old_start + offset)
        new_content = apply_hunk(content, hunk_lines, adjusted_start)
        offset += len(new_content) - len(content)
        content = new_content

    # Write result
    if is_deleted:
        if os.path.exists(target_path):
            os.remove(target_path)
            files_changed.append(f"deleted: {target_path}")
    else:
        parent_dir = os.path.dirname(target_path)
        if parent_dir:
            os.makedirs(parent_dir, exist_ok=True)
        with open(target_path, 'w') as f:
            f.writelines(content)
        files_changed.append(f"{'created' if is_new_file else 'modified'}: {target_path}")

print(f"Applied patch: {len(files_changed)} file(s) changed")
for change in files_changed:
    print(f"  {change}")
if errors:
    print("Errors:")
    for e in errors:
        print(f"  {e}")
PYEOF
```

## Step 4: Report results

```
Applied patch successfully:
  modified: src/main.py
  created: src/new_feature.py
  deleted: src/old_file.py
```

## Handling failures

If a hunk fails to apply (context lines don't match), report:
```
error: patch failed: <path>:<line-number>
error: <path>: patch does not apply
```

Common causes and fixes:
- **Context mismatch**: the file has been modified since the patch was created — edit the patch's context lines to match, or apply manually
- **Already applied**: the change is already in the file — check with `agents/patch.md` (diff working tree vs previous state)
- **Wrong base**: the patch was made against a different branch — checkout the right branch first

## Apply with --reverse (revert a patch)

To undo a previously applied patch, swap `+` and `-` lines and reverse the file order:

```bash
python3 -c "
lines = open('PATCH_FILE').readlines()
result = []
for line in lines:
    if line.startswith('+') and not line.startswith('+++'):
        result.append('-' + line[1:])
    elif line.startswith('-') and not line.startswith('---'):
        result.append('+' + line[1:])
    elif line.startswith('--- '):
        result.append('+++ ' + line[4:])
    elif line.startswith('+++ '):
        result.append('--- ' + line[4:])
    else:
        result.append(line)
open('reversed.patch', 'w').writelines(result)
print('Reversed patch written to reversed.patch')
"
```

Then apply `reversed.patch` using the same steps above.
