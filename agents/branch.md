# Agent: git branch / checkout

List branches, create a new branch, or switch to an existing branch.

## List branches

```bash
# List all branch names
ls .git/refs/heads/

# Current branch
cat .git/HEAD
```

Format output: prefix current branch with `*`, others with `  `.

```
* main
  feature/login
  fix/bug-42
```

## Create a new branch (without switching)

1. Get current HEAD commit SHA:
```bash
python3 -c "
import os
head = open('.git/HEAD').read().strip()
if head.startswith('ref: '):
    ref = '.git/' + head[5:]
    print(open(ref).read().strip() if os.path.exists(ref) else '')
else:
    print(head)
"
```

2. Write the SHA to the new branch ref file:

Write to `.git/refs/heads/<new-branch-name>`:
```
<commit-sha>
```
(include trailing newline)

3. Confirm: `Branch '<name>' created at <short-sha>`

## Switch to an existing branch (checkout)

### 1. Verify branch exists

```bash
cat .git/refs/heads/<branch-name>
```

If file doesn't exist: `error: branch '<name>' does not exist`

### 2. Get the target tree

```bash
python3 << 'PYEOF'
import zlib

def read_obj(sha):
    data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    return data[:null].decode().split()[0], data[null+1:]

commit_sha = open('.git/refs/heads/BRANCH_NAME').read().strip()
_, body = read_obj(commit_sha)
for line in body.decode().split('\n'):
    if line.startswith('tree '):
        print(line[5:])
        break
PYEOF
```

### 3. Walk the tree to collect all files

```bash
python3 << 'PYEOF'
import zlib

def read_obj(sha):
    data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    return data[:null].decode().split()[0], data[null+1:]

def walk_tree(tree_sha, prefix=''):
    _, body = read_obj(tree_sha)
    i = 0
    while i < len(body):
        sp = body.index(b' ', i)
        null = body.index(b'\x00', sp)
        mode = body[i:sp].decode()
        name = body[sp+1:null].decode()
        entry_sha = body[null+1:null+21].hex()
        path = f"{prefix}/{name}" if prefix else name
        if mode in ('40000', '040000'):
            walk_tree(entry_sha, path)
        else:
            _, content = read_obj(entry_sha)
            print(f"{mode}\t{path}\t{entry_sha}")
        i = null + 21

walk_tree("TREE_SHA")
PYEOF
```

This outputs lines like `100644\tREADME.md\tabc123...`

### 4. Write each file to the working directory

```bash
python3 << 'PYEOF'
import zlib, os

# Paste the entries from step 3 here as a list
file_entries = [
    ("100644", "README.md", "abc123..."),
    # ...
]

def read_blob(sha):
    data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    return data[null+1:]

for mode, path, sha in file_entries:
    content = read_blob(sha)
    parent = os.path.dirname(path)
    if parent:
        os.makedirs(parent, exist_ok=True)
    with open(path, 'wb') as f:
        f.write(content)
    if mode == '100755':
        os.chmod(path, 0o755)

print(f"Checked out {len(file_entries)} files")
PYEOF
```

### 5. Update HEAD

Write to `.git/HEAD`:
```
ref: refs/heads/<branch-name>
```
(include trailing newline)

### 6. Confirm

```
Switched to branch '<branch-name>'
```

## Create and switch (checkout -b)

1. Create the branch (pointing at current HEAD commit SHA)
2. Update `.git/HEAD` to `ref: refs/heads/<new-branch-name>`
3. No working directory changes needed — files are already in the right state

```
Switched to a new branch '<name>'
```

## Delete a branch

1. Check the user is not on the branch being deleted (read `.git/HEAD`)
2. Delete the file: `.git/refs/heads/<branch-name>`
3. Confirm: `Deleted branch '<name>'`
