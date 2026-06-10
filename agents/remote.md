# Agent: git remote

Add, remove, rename, or list remote repositories. All operations read/write `.git/config` directly.

## List remotes

```bash
cat .git/config
```

Parse and display all `[remote "..."]` sections:

```bash
python3 -c "
import re
cfg = open('.git/config').read()
remotes = re.findall(r'\[remote \"([^\"]+)\"\].*?\n\s*url\s*=\s*([^\n]+)', cfg, re.DOTALL)
for name, url in remotes:
    print(f'{name}\t{url.strip()}')
"
```

For `remote -v`, show both fetch and push URL (same unless overridden):
```
origin  git@github.com:owner/repo.git (fetch)
origin  git@github.com:owner/repo.git (push)
```

## Add a remote

Append a new section to `.git/config`:

```bash
python3 << 'PYEOF'
NAME = "REPLACE_NAME"   # e.g. "origin"
URL  = "REPLACE_URL"    # e.g. "git@github.com:owner/repo.git"

new_section = f'\n[remote "{NAME}"]\n\turl = {URL}\n\tfetch = +refs/heads/*:refs/remotes/{NAME}/*\n'

with open('.git/config', 'a') as f:
    f.write(new_section)
print(f"Added remote '{NAME}' -> {URL}")
PYEOF
```

Verify the remote doesn't already exist before adding:
```bash
grep -q "remote \"$NAME\"" .git/config && echo "EXISTS" || echo "OK"
```

## Remove a remote

Delete the `[remote "<name>"]` section and all its indented config lines:

```bash
python3 << 'PYEOF'
import re
NAME = "REPLACE_NAME"

with open('.git/config') as f:
    cfg = f.read()

# Remove the section block: from [remote "name"] to the next [section] or EOF
pattern = rf'\n?\[remote "{re.escape(NAME)}"\][^\[]*'
new_cfg = re.sub(pattern, '', cfg)

if new_cfg == cfg:
    print(f"error: No such remote: '{NAME}'")
else:
    with open('.git/config', 'w') as f:
        f.write(new_cfg)
    print(f"Removed remote '{NAME}'")
PYEOF
```

Also clean up any tracking refs:
```bash
rm -f .git/refs/remotes/<name>/*
rmdir .git/refs/remotes/<name> 2>/dev/null || true
```

## Rename a remote

```bash
python3 << 'PYEOF'
OLD_NAME = "REPLACE_OLD"
NEW_NAME = "REPLACE_NEW"

with open('.git/config') as f:
    cfg = f.read()

# Replace section header and fetch refspec
cfg = cfg.replace(f'[remote "{OLD_NAME}"]', f'[remote "{NEW_NAME}"]')
cfg = cfg.replace(f'refs/remotes/{OLD_NAME}/', f'refs/remotes/{NEW_NAME}/')

with open('.git/config', 'w') as f:
    f.write(cfg)

print(f"Renamed remote '{OLD_NAME}' -> '{NEW_NAME}'")
PYEOF
```

Move tracking refs:
```bash
if [ -d .git/refs/remotes/<old-name> ]; then
    mv .git/refs/remotes/<old-name> .git/refs/remotes/<new-name>
fi
```

## Change remote URL

```bash
python3 << 'PYEOF'
import re
NAME    = "REPLACE_NAME"
NEW_URL = "REPLACE_URL"

with open('.git/config') as f:
    cfg = f.read()

# Find [remote "NAME"] section and replace its url line
pattern = rf'(\[remote "{re.escape(NAME)}"\][^\[]*?url\s*=\s*)([^\n]+)'
new_cfg = re.sub(pattern, lambda m: m.group(1) + NEW_URL, cfg, flags=re.DOTALL)

if new_cfg == cfg:
    print(f"error: no remote '{NAME}' or url line not found")
else:
    with open('.git/config', 'w') as f:
        f.write(new_cfg)
    print(f"Updated remote '{NAME}' URL -> {NEW_URL}")
PYEOF
```

## Show remote details

```bash
python3 << 'PYEOF'
import re
NAME = "REPLACE_NAME"

cfg = open('.git/config').read()
# Extract the remote section
m = re.search(rf'\[remote "{re.escape(NAME)}"\]([^\[]*)', cfg)
if not m:
    print(f"error: no such remote: '{NAME}'")
else:
    section = m.group(1)
    url_m = re.search(r'url\s*=\s*(.+)', section)
    url = url_m.group(1).strip() if url_m else '(none)'
    
    # Check remote tracking refs
    import os
    remote_ref_dir = f'.git/refs/remotes/{NAME}'
    branches = os.listdir(remote_ref_dir) if os.path.isdir(remote_ref_dir) else []
    
    print(f"* remote {NAME}")
    print(f"  Fetch URL: {url}")
    print(f"  Push  URL: {url}")
    print(f"  Remote branches:")
    for b in sorted(branches):
        sha = open(f'{remote_ref_dir}/{b}').read().strip()
        print(f"    {b} ({sha[:7]})")
PYEOF
```
