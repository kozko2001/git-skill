# Agent: git log

Show the commit history of the current branch (or a specified branch/SHA).

## Steps

### 1. Resolve HEAD to a commit SHA

```bash
cat .git/HEAD
```

- If output is `ref: refs/heads/<branch>`, read `.git/refs/heads/<branch>` for the SHA.
- If the ref file doesn't exist yet, report "No commits yet on branch `<branch>`" and stop.
- If output is a bare 40-char SHA (detached HEAD), use it directly.

### 2. Determine options

From the user's request, determine:
- `max_commits`: default 20, or whatever number they specified (e.g. `git log -5`)
- `oneline`: true if user asked for `--oneline` or brief format
- `start_sha`: branch-specific if user specified one, otherwise HEAD

### 3. Walk the commit chain

Run this Python block, replacing `START_SHA` and `MAX` with actual values:

```bash
python3 << 'PYEOF'
import zlib, os, datetime

def read_obj(sha):
    path = f".git/objects/{sha[:2]}/{sha[2:]}"
    data = zlib.decompress(open(path, 'rb').read())
    null = data.index(b'\x00')
    return data[:null].decode().split()[0], data[null+1:]

def parse_commit(body):
    text = body.decode('utf-8', errors='replace')
    lines = text.split('\n')
    info = {'parents': [], 'author': '', 'tree': '', 'message': ''}
    msg_idx = 0
    for i, line in enumerate(lines):
        if line.startswith('tree '):     info['tree'] = line[5:]
        elif line.startswith('parent '): info['parents'].append(line[7:])
        elif line.startswith('author '): info['author'] = line[7:]
        elif line == '':
            msg_idx = i + 1
            break
    info['message'] = '\n'.join(lines[msg_idx:]).strip()
    return info

def format_author_date(author_line):
    # "Name <email> timestamp tz"
    parts = author_line.rsplit(' ', 2)
    name_email = parts[0] if len(parts) >= 3 else author_line
    ts = int(parts[1]) if len(parts) >= 3 else 0
    date_str = datetime.datetime.fromtimestamp(ts).strftime('%a %b %d %H:%M:%S %Y %z')
    return name_email, date_str

sha = "START_SHA"
max_commits = MAX
count = 0
oneline = ONELINE  # True or False

while sha and count < max_commits:
    obj_type, body = read_obj(sha)
    info = parse_commit(body)
    name_email, date_str = format_author_date(info['author'])

    if oneline:
        short = sha[:7]
        first_line = info['message'].split('\n')[0]
        print(f"{short} {first_line}")
    else:
        print(f"commit {sha}")
        print(f"Author: {name_email}")
        print(f"Date:   {date_str}")
        print()
        for line in info['message'].split('\n'):
            print(f"    {line}")
        print()

    sha = info['parents'][0] if info['parents'] else None
    count += 1
PYEOF
```

Replace:
- `START_SHA` → the resolved commit SHA (in quotes)
- `MAX` → integer limit (e.g. `20`)
- `ONELINE` → `True` or `False`

### 4. Show branch context

Before the log output, show:
```
On branch <branch-name>  (or "HEAD detached at <sha>")
```

### 5. Handle merge commits

If a commit has multiple `parent` entries, it is a merge commit. Show all parents:
```
Merge: <sha1[:7]> <sha2[:7]>
```

Follow only the first parent (main line) for the log traversal.
