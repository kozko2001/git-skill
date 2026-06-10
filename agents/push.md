# Agent: git push

Push local commits to a remote repository. Supports two transport methods:
- **GitHub API** (HTTPS, requires `GITHUB_TOKEN`)
- **SSH git protocol** (requires SSH access to the remote, uses `ssh` binary)

## Step 0: Read repo config

```bash
cat .git/config
cat .git/HEAD
```

Parse the remote URL and determine the current branch. See `references/git-internals.md` for the `parse_remote_url` and `url_to_owner_repo` helper patterns.

## Step 1: Choose transport method

- If `$GITHUB_TOKEN` is set → use **GitHub API**
- If remote URL starts with `git@` → use **SSH git protocol**
- If HTTPS URL and no token → ask user for one

---

## Method A: GitHub API (HTTPS / token)

### A1. Get local and remote state

```bash
# Local branch HEAD
LOCAL_SHA=$(cat .git/refs/heads/<branch>)

# Remote branch HEAD via API
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/git/refs/heads/<branch>
```

If the remote ref doesn't exist (404), this is the first push to that branch.

### A2. Find commits to push

```bash
python3 << 'PYEOF'
import zlib, os

def read_commit(sha):
    data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    body = data[null+1:].decode('utf-8', errors='replace')
    info = {}
    for line in body.split('\n'):
        if line.startswith('tree '):   info['tree'] = line[5:]
        elif line.startswith('parent '): info.setdefault('parents', []).append(line[7:])
    info.setdefault('parents', [])
    return info

# Walk back from LOCAL_SHA until we hit REMOTE_SHA (or run out of history)
local_sha = "LOCAL_SHA"
remote_sha = "REMOTE_SHA"  # empty string if first push

to_push = []
sha = local_sha
while sha and sha != remote_sha:
    to_push.append(sha)
    info = read_commit(sha)
    sha = info['parents'][0] if info['parents'] else None

to_push.reverse()  # oldest first
for s in to_push:
    print(s)
PYEOF
```

### A3. Upload each commit's objects

For each commit SHA (oldest first):

```bash
python3 << 'PYEOF'
import zlib, json, subprocess, os, base64

TOKEN = os.environ.get('GITHUB_TOKEN', '')
OWNER = "REPLACE_OWNER"
REPO  = "REPLACE_REPO"
API   = f"https://api.github.com/repos/{OWNER}/{REPO}/git"

def gh_post(endpoint, payload):
    cmd = [
        'curl', '-s', '-X', 'POST',
        '-H', f'Authorization: Bearer {TOKEN}',
        '-H', 'Content-Type: application/json',
        f'{API}/{endpoint}',
        '-d', json.dumps(payload)
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    return json.loads(result.stdout)

def read_obj(sha):
    data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    header = data[:null].decode()
    return header.split()[0], data[null+1:]

def walk_tree(tree_sha, prefix=''):
    _, body = read_obj(tree_sha)
    i = 0
    files = {}
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

# Replace with actual commit SHAs to push (oldest first)
COMMITS = ["COMMIT_SHA_1", "COMMIT_SHA_2"]
BRANCH  = "REPLACE_BRANCH"
REMOTE_PARENT_SHA = "REPLACE_REMOTE_SHA"  # current remote HEAD, or empty

prev_sha = REMOTE_PARENT_SHA
for commit_sha in COMMITS:
    _, cbody = read_obj(commit_sha)
    ctext = cbody.decode('utf-8', errors='replace')
    clines = {l.split(' ')[0]: ' '.join(l.split(' ')[1:]) for l in ctext.split('\n') if l}

    tree_sha = ctext.split('\n')[0].split(' ')[1]
    files = walk_tree(tree_sha)

    # Upload blobs
    gh_tree_entries = []
    for path, (mode, blob_sha) in files.items():
        _, blob_content = read_obj(blob_sha)
        resp = gh_post('blobs', {
            'content': base64.b64encode(blob_content).decode(),
            'encoding': 'base64'
        })
        gh_tree_entries.append({
            'path': path,
            'mode': mode if mode != '40000' else '040000',
            'type': 'blob',
            'sha': resp['sha']
        })

    # Create tree
    tree_payload = {'tree': gh_tree_entries}
    if prev_sha:
        tree_payload['base_tree'] = prev_sha  # reuse parent tree as base
    tree_resp = gh_post('trees', tree_payload)

    # Parse commit metadata
    import re, datetime
    author_m = re.search(r'author (.+) <(.+)> (\d+)', ctext)
    msg_start = ctext.index('\n\n') + 2
    message = ctext[msg_start:].strip()

    author_name  = author_m.group(1) if author_m else 'Unknown'
    author_email = author_m.group(2) if author_m else 'unknown@example.com'
    ts = int(author_m.group(3)) if author_m else 0
    date_iso = datetime.datetime.utcfromtimestamp(ts).strftime('%Y-%m-%dT%H:%M:%SZ')

    # Create commit
    commit_payload = {
        'message': message,
        'tree': tree_resp['sha'],
        'author': {'name': author_name, 'email': author_email, 'date': date_iso}
    }
    if prev_sha:
        commit_payload['parents'] = [prev_sha]

    commit_resp = gh_post('commits', commit_payload)
    prev_sha = commit_resp['sha']
    print(f"Pushed commit: {commit_resp['sha'][:7]} - {message.split(chr(10))[0]}")

# Update or create the remote ref
import subprocess, json
final_sha = prev_sha
patch_cmd = [
    'curl', '-s', '-X', 'PATCH',
    '-H', f'Authorization: Bearer {TOKEN}',
    '-H', 'Content-Type: application/json',
    f'{API}/refs/heads/{BRANCH}',
    '-d', json.dumps({'sha': final_sha, 'force': False})
]
result = subprocess.run(patch_cmd, capture_output=True, text=True)
resp = json.loads(result.stdout)
if resp.get('ref'):
    print(f"Updated refs/heads/{BRANCH} -> {final_sha[:7]}")
else:
    # Branch doesn't exist yet, create it
    post_cmd = [
        'curl', '-s', '-X', 'POST',
        '-H', f'Authorization: Bearer {TOKEN}',
        '-H', 'Content-Type: application/json',
        f'{API}/refs',
        '-d', json.dumps({'ref': f'refs/heads/{BRANCH}', 'sha': final_sha})
    ]
    subprocess.run(post_cmd)
    print(f"Created refs/heads/{BRANCH} -> {final_sha[:7]}")
PYEOF
```

---

## Method B: SSH git protocol

Uses the `ssh` binary (allowed) to communicate with the remote using the git smart protocol. **Never uses the `git` binary.**

### B1. Parse SSH remote URL

From `.git/config`, extract remote URL like `git@github.com:owner/repo.git`.

```bash
python3 -c "
import re
cfg = open('.git/config').read()
m = re.search(r'url\s*=\s*(git@[^\n]+)', cfg)
if m:
    url = m.group(1).strip()
    host, path = url.split(':', 1)
    host = host.split('@', 1)[1]
    print('HOST:', host)
    print('PATH:', path.rstrip('.git').rstrip('/'))
    print('REPO_PATH:', path)
"
```

### B2. Create the PACK file

The PACK file contains all git objects to transfer (blobs, trees, commits).

```bash
python3 << 'PYEOF'
import zlib, hashlib, struct, os

def read_obj(sha):
    data = zlib.decompress(open(f'.git/objects/{sha[:2]}/{sha[2:]}', 'rb').read())
    null = data.index(b'\x00')
    return data[:null].decode().split()[0], data[null+1:]

TYPE_MAP = {'commit': 1, 'tree': 2, 'blob': 3, 'tag': 4}

def encode_obj_header(obj_type, size):
    type_num = TYPE_MAP[obj_type]
    # First byte: 3-bit type in bits 6-4, 4 LSBs of size in bits 3-0, bit 7 = MSB continuation
    first = (type_num << 4) | (size & 0xF)
    size >>= 4
    result = bytearray()
    while size:
        result.append(first | 0x80)
        first = size & 0x7F
        size >>= 7
    result.append(first)
    return bytes(result)

def collect_objects(commit_sha, stop_sha=None):
    """Collect all unique objects reachable from commit_sha, stopping at stop_sha."""
    seen = set()
    queue = [commit_sha]
    if stop_sha:
        # Pre-populate with objects already on remote (approximation: just the commit)
        seen.add(stop_sha)

    while queue:
        sha = queue.pop()
        if sha in seen: continue
        seen.add(sha)
        obj_type, body = read_obj(sha)
        if obj_type == 'commit':
            lines = body.decode().split('\n')
            for line in lines:
                if line.startswith('tree '): queue.append(line[5:])
                elif line.startswith('parent '): queue.append(line[7:])
        elif obj_type == 'tree':
            i = 0
            while i < len(body):
                sp = body.index(b' ', i)
                null = body.index(b'\x00', sp)
                entry_sha = body[null+1:null+21].hex()
                queue.append(entry_sha)
                i = null + 21
    return seen

def build_pack(object_shas):
    objects_data = b''
    count = 0
    for sha in object_shas:
        obj_type, content = read_obj(sha)
        header = encode_obj_header(obj_type, len(content))
        objects_data += header + zlib.compress(content)
        count += 1

    pack = b'PACK'
    pack += struct.pack('>I', 2)       # version
    pack += struct.pack('>I', count)   # object count
    pack += objects_data
    pack += hashlib.sha1(pack).digest()
    return pack

# Replace with actual values
LOCAL_SHA  = "REPLACE_LOCAL_SHA"
REMOTE_SHA = "REPLACE_REMOTE_SHA"  # or empty string

objects = collect_objects(LOCAL_SHA, REMOTE_SHA if REMOTE_SHA else None)
pack_data = build_pack(objects)

with open('/tmp/push.pack', 'wb') as f:
    f.write(pack_data)
print(f"Pack file: /tmp/push.pack ({len(pack_data)} bytes, {len(objects)} objects)")
PYEOF
```

### B3. Run the git smart protocol over SSH

```bash
python3 << 'PYEOF'
import subprocess, struct

REPO_PATH  = "REPLACE_REPO_PATH"   # e.g. "owner/repo.git"
SSH_HOST   = "github.com"
LOCAL_SHA  = "REPLACE_LOCAL_SHA"
REMOTE_SHA = "REPLACE_REMOTE_SHA"  # 40 zeros if new branch
BRANCH     = "REPLACE_BRANCH"
PACK_FILE  = "/tmp/push.pack"

def pkt_line(data):
    if data is None:
        return b'0000'
    encoded = data.encode() if isinstance(data, str) else data
    length = len(encoded) + 4
    return f"{length:04x}".encode() + encoded

def build_push_payload(old_sha, new_sha, branch, pack_path):
    payload = b''
    ref_line = f"{old_sha} {new_sha} refs/heads/{branch}\x00 side-band-64k agent=git-skill/1.0\n"
    payload += pkt_line(ref_line)
    payload += b'0000'  # flush
    with open(pack_path, 'rb') as f:
        payload += f.read()
    return payload

zero_sha = '0' * 40
old_sha = REMOTE_SHA if REMOTE_SHA else zero_sha

payload = build_push_payload(old_sha, LOCAL_SHA, BRANCH, PACK_FILE)

# Send to remote via SSH
proc = subprocess.Popen(
    ['ssh', SSH_HOST, f'git-receive-pack \'{REPO_PATH}\''],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)
stdout, stderr = proc.communicate(input=payload, timeout=30)

# Parse response
if proc.returncode == 0:
    print("Push successful")
    # Parse server response (pkt-lines)
    i = 0
    while i < len(stdout):
        if stdout[i:i+4] == b'0000':
            break
        length = int(stdout[i:i+4], 16)
        if length == 0: break
        line = stdout[i+4:i+length].decode('utf-8', errors='replace').strip()
        print(f"  remote: {line}")
        i += length
else:
    print(f"Push failed: {stderr.decode()}")
PYEOF
```

---

## After push

Store the new remote ref locally to track what the remote has:

```bash
mkdir -p .git/refs/remotes/origin
echo "<new-sha>" > .git/refs/remotes/origin/<branch>
```

Report:
```
To git@github.com:owner/repo.git
   abc1234..def5678  main -> main
```
