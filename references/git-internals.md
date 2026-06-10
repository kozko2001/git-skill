# Git Internals Reference

## Object store layout

```
.git/
├── HEAD                        # "ref: refs/heads/main" or bare SHA
├── config                      # INI-format: remotes, branches, user
├── description
├── refs/
│   ├── heads/<branch>          # plain text: 40-char commit SHA
│   └── tags/<tag>
└── objects/
    ├── <2-char-prefix>/        # e.g. "ab/"
    │   └── <38-char-suffix>    # e.g. "cdef1234..."
    └── pack/                   # packed objects (advanced, ignore for now)
```

## Object format (inside zlib compression)

```
<type> <size>\x00<content>
```

- **type**: `blob`, `tree`, `commit`, or `tag`
- **size**: decimal byte count of `<content>`
- `\x00`: null byte separator

## Object types

### blob
Raw file content bytes.

### tree
Binary entries, each: `<mode> <name>\x00<20-byte-sha>`
- Entries must be sorted: files by name, directories by name + "/"
- Modes: `100644` (file), `100755` (executable), `40000` (directory), `120000` (symlink)

### commit
Plain text:
```
tree <tree-sha>
parent <parent-sha>
author Name <email> <unix-ts> <tz>
committer Name <email> <unix-ts> <tz>

<message>
```
`parent` line is omitted for the very first commit.

## Core Python building blocks

### Read any object
```python
import zlib

def read_obj(sha, git_dir='.git'):
    path = f"{git_dir}/objects/{sha[:2]}/{sha[2:]}"
    data = zlib.decompress(open(path, 'rb').read())
    null = data.index(b'\x00')
    header = data[:null].decode()       # e.g. "commit 342"
    obj_type = header.split()[0]        # "commit", "blob", "tree"
    return obj_type, data[null+1:]      # raw content bytes
```

### Write any object
```python
import zlib, hashlib, os

def write_obj(obj_type, content, git_dir='.git'):
    if isinstance(content, str):
        content = content.encode()
    store = f"{obj_type} {len(content)}\x00".encode() + content
    sha = hashlib.sha1(store).hexdigest()
    out_dir = f"{git_dir}/objects/{sha[:2]}"
    out_path = f"{out_dir}/{sha[2:]}"
    os.makedirs(out_dir, exist_ok=True)
    if not os.path.exists(out_path):
        with open(out_path, 'wb') as f:
            f.write(zlib.compress(store))
    return sha
```

### Parse a tree object
```python
def parse_tree(body):
    """body: raw bytes after the null separator"""
    entries = []
    i = 0
    while i < len(body):
        sp = body.index(b' ', i)
        null = body.index(b'\x00', sp)
        mode = body[i:sp].decode()
        name = body[sp+1:null].decode()
        sha_hex = body[null+1:null+21].hex()
        entries.append((mode, name, sha_hex))
        i = null + 21
    return entries  # [(mode, name, sha_hex), ...]
```

### Build a tree object
```python
def build_tree(entries, git_dir='.git'):
    """entries: [(mode_str, name, sha_hex), ...]"""
    def sort_key(e):
        mode, name, _ = e
        return name + '/' if mode in ('40000', '040000') else name
    entries = sorted(entries, key=sort_key)
    content = b''
    for mode, name, sha_hex in entries:
        content += f"{mode} {name}\x00".encode()
        content += bytes.fromhex(sha_hex)
    return write_obj('tree', content, git_dir)
```

### Walk a tree recursively
```python
def walk_tree(tree_sha, prefix='', git_dir='.git'):
    """Returns dict of {relative_path: (mode, blob_sha)}"""
    _, body = read_obj(tree_sha, git_dir)
    entries = parse_tree(body)
    files = {}
    for mode, name, sha in entries:
        path = f"{prefix}/{name}" if prefix else name
        if mode in ('40000', '040000'):
            files.update(walk_tree(sha, path, git_dir))
        else:
            files[path] = (mode, sha)
    return files
```

### Resolve HEAD to commit SHA
```python
def resolve_head(git_dir='.git'):
    head = open(f'{git_dir}/HEAD').read().strip()
    if head.startswith('ref: '):
        ref_path = f"{git_dir}/{head[5:]}"
        if os.path.exists(ref_path):
            return open(ref_path).read().strip() or None
        return None  # branch exists but no commits yet
    return head  # detached HEAD

def current_branch(git_dir='.git'):
    head = open(f'{git_dir}/HEAD').read().strip()
    if head.startswith('ref: refs/heads/'):
        return head[len('ref: refs/heads/'):]
    return None  # detached HEAD
```

### Parse .git/config remote URL
```python
import re

def parse_remote_url(config_text, remote='origin'):
    # Find [remote "origin"] section
    pattern = rf'\[remote "{remote}"\].*?\n\s*url\s*=\s*(.+)'
    m = re.search(pattern, config_text, re.DOTALL)
    if not m:
        return None
    url = m.group(1).strip().split('\n')[0].strip()
    return url

def url_to_owner_repo(url):
    # SSH: git@github.com:owner/repo.git
    m = re.match(r'git@github\.com:([^/]+)/(.+?)(?:\.git)?$', url)
    if m:
        return m.group(1), m.group(2)
    # HTTPS: https://github.com/owner/repo.git
    m = re.match(r'https://github\.com/([^/]+)/(.+?)(?:\.git)?$', url)
    if m:
        return m.group(1), m.group(2)
    return None, None
```

## SHA addressing

Given SHA `ab1234...` (40 hex chars):
- Directory: `.git/objects/ab/`
- File: `.git/objects/ab/1234...` (remaining 38 chars)
