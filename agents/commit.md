# Agent: git commit

Create a new commit from the current working tree state.

## Before starting

Ask the user (if not already provided):
1. **Commit message** — required
2. **Files to commit** — default is all files in the working directory; user can specify a subset
3. **Author name and email** — read from `.git/config` first; ask only if not configured

Read author from config:
```bash
python3 -c "
import re
try:
    cfg = open('.git/config').read()
    n = re.search(r'name\s*=\s*(.+)', cfg)
    e = re.search(r'email\s*=\s*(.+)', cfg)
    print('NAME:', n.group(1).strip() if n else '')
    print('EMAIL:', e.group(1).strip() if e else '')
except: print('NAME:\nEMAIL:')
"
```

## Steps

### 1. Collect files to commit

Either use a user-specified list, or scan the working tree (excluding `.git/`):

```bash
find . -not -path './.git/*' -not -name '.git' -type f | sort
```

If the repo has a `.gitignore`, read it and filter accordingly. For simplicity, honor only the patterns listed in `.gitignore` at the root.

### 2. Create blob objects for all files

Run this Python block (builds blobs + full tree in one pass):

```bash
python3 << 'PYEOF'
import zlib, hashlib, os, sys, time, re

GIT_DIR = '.git'
FILES = None  # replace with list of paths, or None for all

def write_obj(obj_type, content):
    if isinstance(content, str):
        content = content.encode()
    store = f"{obj_type} {len(content)}\x00".encode() + content
    sha = hashlib.sha1(store).hexdigest()
    out_dir = f"{GIT_DIR}/objects/{sha[:2]}"
    out_path = f"{out_dir}/{sha[2:]}"
    os.makedirs(out_dir, exist_ok=True)
    if not os.path.exists(out_path):
        with open(out_path, 'wb') as f:
            f.write(zlib.compress(store))
    return sha

def build_tree_for_dir(dir_path, ignore_root_git=True):
    entries = []
    try:
        items = sorted(os.listdir(dir_path))
    except PermissionError:
        return None
    for name in items:
        if ignore_root_git and name == '.git':
            continue
        full = os.path.join(dir_path, name)
        if os.path.islink(full):
            target = os.readlink(full).encode()
            sha = write_obj('blob', target)
            entries.append(('120000', name, sha))
        elif os.path.isfile(full):
            with open(full, 'rb') as f:
                content = f.read()
            sha = write_obj('blob', content)
            mode = '100755' if os.access(full, os.X_OK) else '100644'
            entries.append((mode, name, sha))
        elif os.path.isdir(full):
            sub_sha = build_tree_for_dir(full, ignore_root_git=False)
            if sub_sha:
                entries.append(('40000', name, sub_sha))
    if not entries:
        return None

    def sort_key(e):
        mode, name, _ = e
        return name + '/' if mode in ('40000', '040000') else name

    entries.sort(key=sort_key)
    content = b''
    for mode, name, sha in entries:
        content += f"{mode} {name}\x00".encode()
        content += bytes.fromhex(sha)
    return write_obj('tree', content)

# Build the tree
tree_sha = build_tree_for_dir('.')
print(f"TREE:{tree_sha}")

# Get parent SHA
head = open(f'{GIT_DIR}/HEAD').read().strip()
parent_sha = None
if head.startswith('ref: '):
    ref_path = f"{GIT_DIR}/{head[5:]}"
    if os.path.exists(ref_path):
        parent_sha = open(ref_path).read().strip() or None
else:
    parent_sha = head if len(head) == 40 else None

print(f"PARENT:{parent_sha or ''}")
print(f"BRANCH:{head[len('ref: refs/heads/'):] if head.startswith('ref: refs/heads/') else 'DETACHED'}")
PYEOF
```

Note the `TREE:`, `PARENT:`, and `BRANCH:` values from the output.

### 3. Build and store the commit object

```bash
python3 << 'PYEOF'
import zlib, hashlib, os, time

GIT_DIR = '.git'

def write_obj(obj_type, content):
    if isinstance(content, str):
        content = content.encode()
    store = f"{obj_type} {len(content)}\x00".encode() + content
    sha = hashlib.sha1(store).hexdigest()
    out_dir = f"{GIT_DIR}/objects/{sha[:2]}"
    os.makedirs(out_dir, exist_ok=True)
    out_path = f"{out_dir}/{sha[2:]}"
    if not os.path.exists(out_path):
        with open(out_path, 'wb') as f:
            f.write(zlib.compress(store))
    return sha

# Replace these with actual values
TREE_SHA   = "REPLACE_TREE_SHA"
PARENT_SHA = "REPLACE_PARENT_SHA"   # empty string if first commit
AUTHOR_NAME  = "REPLACE_NAME"
AUTHOR_EMAIL = "REPLACE_EMAIL"
MESSAGE    = "REPLACE_MESSAGE"
BRANCH     = "REPLACE_BRANCH"       # e.g. "main"

ts = int(time.time())
tz = "+0000"
author_str = f"{AUTHOR_NAME} <{AUTHOR_EMAIL}> {ts} {tz}"

lines = [f"tree {TREE_SHA}"]
if PARENT_SHA:
    lines.append(f"parent {PARENT_SHA}")
lines += [
    f"author {author_str}",
    f"committer {author_str}",
    "",
    MESSAGE
]
commit_content = "\n".join(lines)
commit_sha = write_obj('commit', commit_content)

# Update branch ref
if BRANCH != 'DETACHED':
    ref_path = f"{GIT_DIR}/refs/heads/{BRANCH}"
    os.makedirs(os.path.dirname(ref_path), exist_ok=True)
    with open(ref_path, 'w') as f:
        f.write(commit_sha + '\n')
else:
    with open(f'{GIT_DIR}/HEAD', 'w') as f:
        f.write(commit_sha + '\n')

print(f"[{BRANCH} {commit_sha[:7]}] {MESSAGE.split(chr(10))[0]}")
PYEOF
```

### 4. Report success

```
[main a1b2c3d] Your commit message
 3 files changed
```

Show the number of files that differ from the parent (or all files if first commit).

## Handling first commit (no parent)

When `PARENT_SHA` is empty, omit the `parent` line entirely from the commit object. The tree is still required.

## Files specified by user

If the user said "commit only `src/` and `README.md`", use only those paths. Build blobs only for those files, and build a tree that mirrors only that subset. For partial commits, be sure to merge with the parent tree to avoid losing other files — read the parent tree, update only the specified entries, and write the new tree.
