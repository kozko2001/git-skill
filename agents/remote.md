# Agent: git remote

Add, remove, rename, or list remotes. All operations read/write `.git/config` — plain text INI. No binary, no `git`, no `python3`.

---

## List remotes

```bash
# Names only
awk '/^\[remote "/{gsub(/.*"/,""); gsub(/"\].*/,""); print}' .git/config

# With URLs (-v style)
awk '/^\[remote "/{name=$0; gsub(/.*"/,"",name); gsub(/"\].*/,"",name)}
     /url *=/{gsub(/.*= */,""); printf "%s\t%s (fetch)\n%s\t%s (push)\n", name, $0, name, $0}' \
    .git/config
```

---

## Add a remote

```bash
NAME="REPLACE_NAME"    # e.g. "origin"
URL="REPLACE_URL"      # e.g. "git@github.com:owner/repo.git"

# Check not already present
grep -q "^\[remote \"$NAME\"\]" .git/config && {
    echo "error: remote '$NAME' already exists"
    exit 1
}

printf '\n[remote "%s"]\n\turl = %s\n\tfetch = +refs/heads/*:refs/remotes/%s/*\n' \
    "$NAME" "$URL" "$NAME" >> .git/config

printf "Added remote '%s' -> %s\n" "$NAME" "$URL"
```

---

## Remove a remote

```bash
NAME="REPLACE_NAME"

grep -q "^\[remote \"$NAME\"\]" .git/config || {
    echo "error: no such remote: '$NAME'"
    exit 1
}

# Remove [remote "NAME"] block: from the section header to the next [ or EOF
awk "
/^\[remote \"$NAME\"\]/ { skip=1; next }
skip && /^\[/ { skip=0 }
!skip { print }
" .git/config > /tmp/config.tmp && mv /tmp/config.tmp .git/config

# Remove tracking refs
rm -f ".git/refs/remotes/$NAME"/*
rmdir ".git/refs/remotes/$NAME" 2>/dev/null || true

printf "Removed remote '%s'\n" "$NAME"
```

---

## Rename a remote

```bash
OLD="REPLACE_OLD"
NEW="REPLACE_NEW"

grep -q "^\[remote \"$OLD\"\]" .git/config || {
    echo "error: no such remote: '$OLD'"
    exit 1
}

sed -i \
    -e "s/^\[remote \"$OLD\"\]/[remote \"$NEW\"]/" \
    -e "s|refs/remotes/$OLD/|refs/remotes/$NEW/|g" \
    .git/config

# Move tracking ref directory
[ -d ".git/refs/remotes/$OLD" ] && mv ".git/refs/remotes/$OLD" ".git/refs/remotes/$NEW"

printf "Renamed remote '%s' -> '%s'\n" "$OLD" "$NEW"
```

---

## Change remote URL

```bash
NAME="REPLACE_NAME"
NEW_URL="REPLACE_URL"

grep -q "^\[remote \"$NAME\"\]" .git/config || {
    echo "error: no such remote: '$NAME'"
    exit 1
}

# Replace the url = line inside the [remote "NAME"] section
awk "
/^\[remote \"$NAME\"\]/ { in_section=1 }
in_section && /^\[/ && !/^\[remote \"$NAME\"\]/ { in_section=0 }
in_section && /url *=/ { print \"\turl = $NEW_URL\"; next }
{ print }
" .git/config > /tmp/config.tmp && mv /tmp/config.tmp .git/config

printf "Updated remote '%s' URL -> %s\n" "$NAME" "$NEW_URL"
```

---

## Show remote details

```bash
NAME="REPLACE_NAME"

grep -q "^\[remote \"$NAME\"\]" .git/config || {
    echo "error: no such remote: '$NAME'"
    exit 1
}

url=$(awk "/^\[remote \"$NAME\"\]/{f=1} f&&/url *=/{gsub(/.*= */,\"\");print;exit}" .git/config)

printf '* remote %s\n' "$NAME"
printf '  Fetch URL: %s\n' "$url"
printf '  Push  URL: %s\n' "$url"

ref_dir=".git/refs/remotes/$NAME"
if [ -d "$ref_dir" ]; then
    printf '  Remote branches:\n'
    for ref in "$ref_dir"/*; do
        branch=$(basename "$ref")
        sha=$(cat "$ref" | tr -d '\n')
        printf '    %s (%s)\n' "$branch" "${sha:0:7}"
    done
fi
```
