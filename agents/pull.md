# Agent: git pull / fetch

Fetch commits from a remote and update the local branch. Supports two transport methods:
- **GitHub API** (HTTPS, requires `GITHUB_TOKEN`)
- **SSH git protocol** (uses `ssh` binary, no `git` binary)

## Step 0: Read repo config

```bash
cat .git/config
cat .git/HEAD
```

Parse remote URL, current branch, and local HEAD SHA.

## Step 1: Choose transport method

- `$GITHUB_TOKEN` set → use **GitHub API**
- Remote URL starts with `git@` → use **SSH git protocol**

---

## Method A: GitHub API

### A1. Get remote branch state

```bash
TOKEN="$GITHUB_TOKEN"
OWNER="replace"
REPO="replace"
BRANCH="replace"

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/git/refs/heads/$BRANCH"
```

Extract the remote commit SHA from `.object.sha` in the response.

If local HEAD == remote SHA: `Already up to date.` — stop.

### A2. Fetch the remote commit chain

Walk remote commits from the remote SHA back to the local SHA (common ancestor). Use:

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/git/commits/$REMOTE_SHA"
```

Response contains `sha`, `message`, `author`, `tree.sha`, `parents[].sha`.

Collect all new commit SHAs (oldest first, stopping when you reach local HEAD or a commit whose SHA matches an existing local object).

### A3. Fetch all tree objects and blobs

For the newest commit's tree, fetch the full recursive tree in one call:

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/$OWNER/$REPO/git/trees/$TREE_SHA?recursive=1"
```

Response contains a `tree` array with entries: `path`, `mode`, `type`, `sha`.

### A4. Download blobs and store locally

```bash
python3 << 'PYEOF'
import subprocess, zlib, hashlib, os, json, base64

TOKEN = os.environ.get('GITHUB_TOKEN', '')
OWNER = "REPLACE_OWNER"
REPO  = "REPLACE_REPO"
API   = f"https://api.github.com/repos/{OWNER}/{REPO}/git"

# tree_entries: list of dicts from the API response
tree_entries = []  # replace with actual API response data

def write_obj(obj_type, content):
    if isinstance(content, str): content = content.encode()
    store = f"{obj_type} {len(content)}\x00".encode() + content
    sha = hashlib.sha1(store).hexdigest()
    out_dir = f".git/objects/{sha[:2]}"
    out_path = f"{out_dir}/{sha[2:]}"
    os.makedirs(out_dir, exist_ok=True)
    if not os.path.exists(out_path):
        with open(out_path, 'wb') as f:
            f.write(zlib.compress(store))
    return sha

def fetch_blob(sha):
    cmd = ['curl', '-s', '-H', f'Authorization: Bearer {TOKEN}',
           f'{API}/blobs/{sha}']
    result = subprocess.run(cmd, capture_output=True, text=True)
    data = json.loads(result.stdout)
    return base64.b64decode(data['content'].replace('\n', ''))

for entry in tree_entries:
    if entry['type'] != 'blob':
        continue
    # Skip if we already have this object
    obj_path = f".git/objects/{entry['sha'][:2]}/{entry['sha'][2:]}"
    if os.path.exists(obj_path):
        continue
    content = fetch_blob(entry['sha'])
    write_obj('blob', content)

print(f"Downloaded {len([e for e in tree_entries if e['type'] == 'blob'])} blobs")
PYEOF
```

### A5. Reconstruct tree objects locally

```bash
python3 << 'PYEOF'
import zlib, hashlib, os, json

# tree_entries from API (list of dicts with path, mode, type, sha)
tree_entries = []  # replace

def write_obj(obj_type, content):
    if isinstance(content, str): content = content.encode()
    store = f"{obj_type} {len(content)}\x00".encode() + content
    sha = hashlib.sha1(store).hexdigest()
    out_dir = f".git/objects/{sha[:2]}"
    os.makedirs(out_dir, exist_ok=True)
    out_path = f"{out_dir}/{sha[2:]}"
    if not os.path.exists(out_path):
        with open(out_path, 'wb') as f:
            f.write(zlib.compress(store))
    return sha

# Build directory structure from flat entry list
from collections import defaultdict
dirs = defaultdict(list)
for entry in tree_entries:
    parts = entry['path'].rsplit('/', 1)
    parent = parts[0] if len(parts) > 1 else ''
    name = parts[-1]
    dirs[parent].append((entry['mode'], name, entry['sha'], entry['type']))

# Build trees bottom-up (deepest directories first)
def get_depth(path): return path.count('/') if path else 0
all_dirs = sorted(dirs.keys(), key=get_depth, reverse=True)

tree_sha_map = {}
for dir_path in all_dirs:
    entries = dirs[dir_path]
    content = b''
    def sort_key(e):
        mode, name, _, etype = e
        return name + '/' if etype == 'tree' else name
    entries.sort(key=sort_key)
    for mode, name, sha, etype in entries:
        actual_sha = tree_sha_map.get(sha, sha)
        content += f"{mode} {name}\x00".encode() + bytes.fromhex(actual_sha)
    tree_sha = write_obj('tree', content)
    tree_sha_map[dir_path] = tree_sha

root_tree_sha = tree_sha_map.get('', '')
print(f"ROOT_TREE:{root_tree_sha}")
PYEOF
```

### A6. Create local commit objects

For each new commit (oldest first):

```bash
python3 << 'PYEOF'
import zlib, hashlib, os, re

# Replace with actual values from the API responses
COMMITS = [
    {
        'sha': 'REMOTE_SHA',
        'tree': 'LOCAL_TREE_SHA',
        'parent': 'PARENT_SHA',     # empty string if none
        'author_name': 'Name',
        'author_email': 'email',
        'date_iso': '2024-01-01T00:00:00Z',
        'message': 'commit message'
    }
]

def write_obj(obj_type, content):
    if isinstance(content, str): content = content.encode()
    store = f"{obj_type} {len(content)}\x00".encode() + content
    sha = hashlib.sha1(store).hexdigest()
    out_dir = f".git/objects/{sha[:2]}"
    os.makedirs(out_dir, exist_ok=True)
    out_path = f"{out_dir}/{sha[2:]}"
    if not os.path.exists(out_path):
        with open(out_path, 'wb') as f:
            f.write(zlib.compress(store))
    return sha

import datetime
for commit in COMMITS:
    # Convert ISO date to unix timestamp
    dt = datetime.datetime.strptime(commit['date_iso'], '%Y-%m-%dT%H:%M:%SZ')
    ts = int(dt.timestamp())
    author_str = f"{commit['author_name']} <{commit['author_email']}> {ts} +0000"
    lines = [f"tree {commit['tree']}"]
    if commit['parent']:
        lines.append(f"parent {commit['parent']}")
    lines += [f"author {author_str}", f"committer {author_str}", "", commit['message']]
    sha = write_obj('commit', '\n'.join(lines))
    print(f"Stored commit: {sha[:7]} - {commit['message'][:50]}")

# Update local branch ref
BRANCH = "REPLACE_BRANCH"
FINAL_SHA = COMMITS[-1]['sha']  # Use the remote SHA as authoritative
# Note: store using the reconstructed local SHA instead if needed
with open(f'.git/refs/heads/{BRANCH}', 'w') as f:
    f.write(FINAL_SHA + '\n')
PYEOF
```

### A7. Update working directory

Use the branch checkout logic from `agents/branch.md` steps 2-4, targeting the new HEAD commit.

---

## Method B: SSH git protocol

### B1. Run git-upload-pack via SSH

```bash
python3 << 'PYEOF'
import subprocess, zlib, hashlib, struct, os

REPO_PATH = "REPLACE_REPO_PATH"   # e.g. "owner/repo.git"
SSH_HOST  = "github.com"
WANT_SHA  = "REPLACE_REMOTE_SHA"
HAVE_SHA  = "REPLACE_LOCAL_SHA"   # empty string if no local commits

def pkt_line(data):
    if data is None:
        return b'0000'
    encoded = data.encode() if isinstance(data, str) else data
    length = len(encoded) + 4
    return f"{length:04x}".encode() + encoded

# Build the upload-pack request
payload = b''
payload += pkt_line(f"want {WANT_SHA} side-band-64k\n")
if HAVE_SHA:
    payload += pkt_line(f"have {HAVE_SHA}\n")
payload += b'0000'
payload += pkt_line("done\n")

proc = subprocess.Popen(
    ['ssh', SSH_HOST, f"git-upload-pack '{REPO_PATH}'"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE
)
stdout, stderr = proc.communicate(input=payload, timeout=60)

if proc.returncode != 0:
    print(f"SSH error: {stderr.decode()}")
    exit(1)

# The response: pkt-line advertisements, then NAK/ACK, then side-band PACK data
# Skip the ref advertisement section (until first flush 0000)
i = 0
while i < len(stdout):
    if stdout[i:i+4] == b'0000':
        i += 4
        break
    length = int(stdout[i:i+4], 16)
    if length == 0: break
    i += length

# Remaining data is sideband-multiplexed PACK
# Sideband: each pkt-line is: length(4) + band(1) + data
# band 1 = pack data, band 2 = progress, band 3 = error
pack_data = b''
while i < len(stdout):
    if stdout[i:i+4] == b'0000':
        break
    length = int(stdout[i:i+4], 16)
    if length < 5: break
    band = stdout[i+4]
    data = stdout[i+5:i+length]
    if band == 1:
        pack_data += data
    elif band == 2:
        print(f"remote: {data.decode('utf-8', errors='replace').strip()}")
    elif band == 3:
        print(f"remote error: {data.decode('utf-8', errors='replace').strip()}")
    i += length

with open('/tmp/fetched.pack', 'wb') as f:
    f.write(pack_data)
print(f"Received pack: {len(pack_data)} bytes")
PYEOF
```

### B2. Unpack the PACK file

```bash
python3 << 'PYEOF'
import zlib, hashlib, struct, os

def write_obj(obj_type, content):
    if isinstance(content, str): content = content.encode()
    store = f"{obj_type} {len(content)}\x00".encode() + content
    sha = hashlib.sha1(store).hexdigest()
    out_dir = f".git/objects/{sha[:2]}"
    os.makedirs(out_dir, exist_ok=True)
    out_path = f"{out_dir}/{sha[2:]}"
    if not os.path.exists(out_path):
        with open(out_path, 'wb') as f:
            f.write(zlib.compress(store))
    return sha

TYPE_NAMES = {1: 'commit', 2: 'tree', 3: 'blob', 4: 'tag'}

with open('/tmp/fetched.pack', 'rb') as f:
    pack = f.read()

# Verify header
assert pack[:4] == b'PACK'
version = struct.unpack('>I', pack[4:8])[0]
count = struct.unpack('>I', pack[8:12])[0]
print(f"Pack version {version}, {count} objects")

pos = 12
stored = []
for _ in range(count):
    # Decode type+size from variable-length header
    byte = pack[pos]; pos += 1
    obj_type_num = (byte >> 4) & 7
    size = byte & 0xF
    shift = 4
    while byte & 0x80:
        byte = pack[pos]; pos += 1
        size |= (byte & 0x7F) << shift
        shift += 7

    type_name = TYPE_NAMES.get(obj_type_num)
    if type_name is None:
        print(f"WARNING: unknown object type {obj_type_num}, skipping")
        break

    # Decompress object data
    decomp = zlib.decompressobj()
    content = decomp.decompress(pack[pos:])
    pos += len(pack[pos:]) - len(decomp.unused_data)

    sha = write_obj(type_name, content)
    stored.append((type_name, sha))
    print(f"  {type_name} {sha[:7]}")

print(f"Unpacked {len(stored)} objects")
PYEOF
```

### B3. Update local refs and working directory

Same as Method A steps A7 onwards: update the branch ref, then update the working directory using the checkout logic from `agents/branch.md`.

---

## After pull

```
From <remote-url>
  abc1234..def5678  main -> origin/main
Already on 'main'
```

Update tracking ref:
```bash
mkdir -p .git/refs/remotes/origin
echo "<remote-sha>" > .git/refs/remotes/origin/<branch>
```
