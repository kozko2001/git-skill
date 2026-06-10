# Agent: git patch (format-patch / diff)

Create a unified diff patch file comparing two states: two commits, a commit vs its parent, or HEAD vs the working tree.

## Determine what to diff

Ask the user (if not already clear):
- **Two commits**: "diff commit A against commit B"
- **Single commit**: "patch for commit X" (diffs X against its parent)
- **Working tree**: "diff uncommitted changes" (compares HEAD tree vs current files on disk)

## Case 1: Diff two commits

### Step 1: Resolve commit SHAs

If user provides branch names or relative refs like `HEAD~2`, resolve them:

```bash
python3 << 'PYEOF'
import os, zlib

def resolve_ref(ref):
    """Resolve a branch name or 'HEAD' to a commit SHA."""
    # Check .git/refs/heads/
    path = f".git/refs/heads/{ref}"
    if os.path.exists(path):
        return open(path).read().strip()
    # Check HEAD
    head = open('.git/HEAD').read().strip()
    if ref == 'HEAD':
        if head.startswith('ref: '):
            rpath = '.git/' + head[5:]
            return open(rpath).read().strip() if os.path.exists(rpath) else None
        return head
    return ref  # assume it's already a SHA

def get_tree(commit_sha):
    data = zlib.decompress(open(f'.git/objects/{commit_sha[:2]}/{commit_sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    body = data[null+1:].decode()
    for line in body.split('\n'):
        if line.startswith('tree '):
            return line[5:]
    return None

sha_a = resolve_ref("REPLACE_SHA_A")  # older commit
sha_b = resolve_ref("REPLACE_SHA_B")  # newer commit

print(f"SHA_A: {sha_a}")
print(f"SHA_B: {sha_b}")
print(f"TREE_A: {get_tree(sha_a)}")
print(f"TREE_B: {get_tree(sha_b)}")
PYEOF
```

### Step 2: Generate the patch

```bash
python3 << 'PYEOF'
import zlib, difflib, os

def read_obj(sha):
    data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    return data[:null].decode().split()[0], data[null+1:]

def walk_tree(tree_sha, prefix=''):
    _, body = read_obj(tree_sha)
    i, files = 0, {}
    while i < len(body):
        sp = body.index(b' ', i)
        null = body.index(b'\x00', sp)
        mode = body[i:sp].decode()
        name = body[sp+1:null].decode()
        entry_sha = body[null+1:null+21].hex()
        path = f"{prefix}/{name}" if prefix else name
        if mode in ('40000', '040000'):
            files.update(walk_tree(entry_sha, path))
        else:
            files[path] = (mode, entry_sha)
        i = null + 21
    return files

TREE_SHA_A = "REPLACE_TREE_A"
TREE_SHA_B = "REPLACE_TREE_B"
OUTPUT_FILE = "changes.patch"  # or user-specified

files_a = walk_tree(TREE_SHA_A)
files_b = walk_tree(TREE_SHA_B)
all_paths = sorted(set(list(files_a.keys()) + list(files_b.keys())))

patch_lines = []
for path in all_paths:
    info_a = files_a.get(path)
    info_b = files_b.get(path)

    if info_a and info_b and info_a[1] == info_b[1]:
        continue  # identical blob SHAs — no change

    if info_a:
        _, content = read_obj(info_a[1])
        try:
            lines_a = content.decode('utf-8').splitlines(keepends=True)
            is_binary_a = False
        except UnicodeDecodeError:
            lines_a = []
            is_binary_a = True
    else:
        lines_a, is_binary_a = [], False

    if info_b:
        _, content = read_obj(info_b[1])
        try:
            lines_b = content.decode('utf-8').splitlines(keepends=True)
            is_binary_b = False
        except UnicodeDecodeError:
            lines_b = []
            is_binary_b = True
    else:
        lines_b, is_binary_b = [], False

    if is_binary_a or is_binary_b:
        patch_lines.append(f"Binary files a/{path} and b/{path} differ\n")
        continue

    diff = list(difflib.unified_diff(
        lines_a, lines_b,
        fromfile=f"a/{path}",
        tofile=f"b/{path}",
        lineterm=''
    ))
    if diff:
        patch_lines.extend(line + '\n' for line in diff)

patch_text = ''.join(patch_lines)
with open(OUTPUT_FILE, 'w') as f:
    f.write(patch_text)
print(f"Patch written to {OUTPUT_FILE} ({len(patch_lines)} lines)")
PYEOF
```

## Case 2: Single commit patch (commit vs its parent)

For a commit with SHA `X`, get its parent SHA, then use Case 1 with `SHA_A = parent`, `SHA_B = X`.

Add the email-style header at the top of the patch file:

```
From <sha> <date>
From: <author name> <email>
Date: <date string>
Subject: [PATCH] <commit message first line>

---
```

## Case 3: Diff working tree vs HEAD

```bash
python3 << 'PYEOF'
import zlib, difflib, os

def read_obj(sha):
    data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    return data[:null].decode().split()[0], data[null+1:]

def walk_tree(tree_sha, prefix=''):
    _, body = read_obj(tree_sha)
    i, files = 0, {}
    while i < len(body):
        sp = body.index(b' ', i)
        null = body.index(b'\x00', sp)
        mode = body[i:sp].decode()
        name = body[sp+1:null].decode()
        entry_sha = body[null+1:null+21].hex()
        path = f"{prefix}/{name}" if prefix else name
        if mode in ('40000', '040000'):
            files.update(walk_tree(entry_sha, path))
        else:
            files[path] = (mode, entry_sha)
        i = null + 21
    return files

# Get HEAD tree
head = open('.git/HEAD').read().strip()
if head.startswith('ref: '):
    commit_sha = open('.git/' + head[5:]).read().strip()
else:
    commit_sha = head

_, cbody = read_obj(commit_sha)
tree_sha = next(l[5:] for l in cbody.decode().split('\n') if l.startswith('tree '))
head_files = walk_tree(tree_sha)

# Scan working tree
working_files = {}
for root, dirs, files in os.walk('.'):
    dirs[:] = [d for d in dirs if d != '.git']
    for f in files:
        full = os.path.join(root, f)
        path = full[2:] if full.startswith('./') else full
        working_files[path] = full

all_paths = sorted(set(list(head_files.keys()) + list(working_files.keys())))
patch_lines = []

for path in all_paths:
    in_head = path in head_files
    on_disk = path in working_files

    if in_head:
        _, blob_content = read_obj(head_files[path][1])
        try:
            lines_a = blob_content.decode('utf-8').splitlines(keepends=True)
        except:
            lines_a = []
    else:
        lines_a = []

    if on_disk:
        with open(working_files[path], 'rb') as fh:
            disk_content = fh.read()
        try:
            lines_b = disk_content.decode('utf-8').splitlines(keepends=True)
        except:
            lines_b = []
    else:
        lines_b = []

    diff = list(difflib.unified_diff(lines_a, lines_b,
                                      fromfile=f"a/{path}", tofile=f"b/{path}", lineterm=''))
    if diff:
        patch_lines.extend(line + '\n' for line in diff)

patch_text = ''.join(patch_lines)
if patch_text:
    with open('working.patch', 'w') as f:
        f.write(patch_text)
    print(f"Patch written to working.patch")
else:
    print("No changes detected")
PYEOF
```

## Output

Report: `Created <filename> (+N lines / -N lines, N files changed)`
