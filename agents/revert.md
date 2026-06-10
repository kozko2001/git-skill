# Agent: git revert

Create a new commit that undoes the changes introduced by a specific commit. Does NOT delete history — it adds a new "undo" commit on top.

## Step 1: Identify the commit to revert

The user can specify:
- A commit SHA (full or abbreviated)
- A relative ref like `HEAD` (revert most recent commit)
- A position in the log like "the 3rd commit"

If abbreviated SHA, resolve it:
```bash
python3 << 'PYEOF'
import os
ABBREV = "REPLACE_SHORT_SHA"  # e.g. "a1b2c3d"

for prefix in os.listdir('.git/objects'):
    if prefix == 'pack' or prefix == 'info': continue
    for obj in os.listdir(f'.git/objects/{prefix}'):
        full = prefix + obj
        if full.startswith(ABBREV):
            print(full)
            break
PYEOF
```

Or use the log agent to show recent commits and let the user identify which one.

## Step 2: Read the target commit and its parent

```bash
python3 << 'PYEOF'
import zlib

def read_obj(sha):
    data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    return data[:null].decode().split()[0], data[null+1:]

TARGET_SHA = "REPLACE_TARGET_SHA"
_, body = read_obj(TARGET_SHA)
text = body.decode('utf-8', errors='replace')

info = {'parents': [], 'tree': None, 'author': '', 'message': ''}
msg_start = 0
for i, line in enumerate(text.split('\n')):
    if line.startswith('tree '):   info['tree'] = line[5:]
    elif line.startswith('parent '): info['parents'].append(line[7:])
    elif line.startswith('author '): info['author'] = line[7:]
    elif line == '':
        msg_start = i + 1
        break

info['message'] = '\n'.join(text.split('\n')[msg_start:]).strip()

print(f"TARGET_TREE: {info['tree']}")
print(f"PARENT_SHA: {info['parents'][0] if info['parents'] else '(none - first commit)'}")
print(f"MESSAGE: {info['message']}")
PYEOF
```

If the commit has no parent (it's the first commit), reverting means clearing all files — confirm with the user before proceeding.

## Step 3: Compute the inverse diff

Compare the target commit's tree against its parent's tree. Find what changed:

```bash
python3 << 'PYEOF'
import zlib

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

TARGET_TREE = "REPLACE_TARGET_TREE"
PARENT_TREE = "REPLACE_PARENT_TREE"

target_files = walk_tree(TARGET_TREE)
parent_files = walk_tree(PARENT_TREE)

all_paths = sorted(set(list(target_files.keys()) + list(parent_files.keys())))

for path in all_paths:
    in_target = path in target_files
    in_parent = path in parent_files

    if in_target and not in_parent:
        print(f"ADDED:{path}:{target_files[path][1]}")  # was added, needs to be removed
    elif not in_target and in_parent:
        print(f"DELETED:{path}:{parent_files[path][1]}")  # was deleted, needs to be restored
    elif target_files[path][1] != parent_files[path][1]:
        print(f"MODIFIED:{path}:{target_files[path][1]}:{parent_files[path][1]}")  # needs restore to parent version
PYEOF
```

## Step 4: Apply the inverse changes to the working directory

For each change identified in Step 3:

**ADDED (was added in target → remove it now):**
```bash
rm <path>
```

**DELETED (was deleted in target → restore it to parent's version):**
```bash
python3 -c "
import zlib
sha = 'PARENT_BLOB_SHA'
data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
null = data.index(b'\x00')
open('PATH_TO_RESTORE', 'wb').write(data[null+1:])
"
```

**MODIFIED (was changed in target → restore to parent's version):**
Same as DELETED — read the parent blob and write it to disk.

## Step 5: Create the revert commit

Use the commit agent logic (from `agents/commit.md`) to:
1. Build blob + tree objects from the current working directory state
2. Create a commit with message: `Revert "<original message>"`
3. Use the current HEAD as the parent

```
Revert "original commit message"

This reverts commit <target-sha>.
```

Full commit message format:
```python
original_message_first_line = "REPLACE_FIRST_LINE"
target_sha = "REPLACE_TARGET_SHA"
message = f'Revert "{original_message_first_line}"\n\nThis reverts commit {target_sha}.'
```

## Step 6: Report

```
[main abc1234] Revert "original commit message"
 2 files changed, 3 insertions(+), 5 deletions(-)
```

## Edge cases

- **Merge commits**: reverting a merge commit requires specifying the mainline parent (`-m 1` equivalent). Ask the user which parent to use as the base.
- **Conflicts**: if the current working tree has changes that overlap with the revert, report the conflicting files and ask the user to resolve them manually before completing the commit.
- **Already reverted**: check if the inverse of the target diff is already present in the current working tree — if so, there's nothing to do.
