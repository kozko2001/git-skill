# Agent: git log

Show commit history. No `git` binary. Uses `gzip` header-swap + `awk`/`grep`.

Read `references/git-internals.md` first for the `git_decompress`, `obj_body`, `first_null_pos` functions.

## Steps

### 1. Resolve HEAD to a commit SHA

```bash
head=$(cat .git/HEAD)
case "$head" in
    ref:\ *)
        ref="${head#ref: }"
        sha=$(cat ".git/$ref" 2>/dev/null | tr -d '\n')
        branch="${ref#refs/heads/}"
        ;;
    *)
        sha="$head"
        branch="HEAD (detached)"
        ;;
esac

[ -z "$sha" ] && echo "No commits yet on branch $branch" && exit 0
printf 'On branch %s\n\n' "$branch"
```

### 2. Set options

```bash
MAX_COMMITS=20          # default; adjust if user said -n <N> or --all
ONELINE=false           # true if user asked for --oneline or brief
```

### 3. Walk the commit chain

```bash
# Source the helper functions from references/git-internals.md
# (paste git_decompress, first_null_pos, obj_body inline here)

git_decompress() {
    local obj="$1"
    local size; size=$(wc -c < "$obj")
    local ds=$(( size - 6 ))
    { printf '\x1f\x8b\x08\x00\x00\x00\x00\x00\x00\x03'
      dd if="$obj" bs=1 skip=2 count="$ds" 2>/dev/null
      printf '\x00\x00\x00\x00\x00\x00\x00\x00'
    } | gzip -d -f -q 2>/dev/null
}

first_null_pos() {
    od -A n -t u1 -v "$1" | tr -s ' \n' '\n' | grep -v '^$' | \
    awk 'NR>0{ if($1+0==0){print NR-1; exit} }'
}

count=0
current_sha="$sha"

while [ -n "$current_sha" ] && [ "$count" -lt "$MAX_COMMITS" ]; do
    obj=".git/objects/${current_sha:0:2}/${current_sha:2}"

    # Decompress to tmpfile
    tmpraw=$(mktemp)
    git_decompress "$obj" > "$tmpraw"

    # Extract body (text after null byte)
    null_pos=$(first_null_pos "$tmpraw")
    tmpbody=$(mktemp)
    dd if="$tmpraw" bs=1 skip=$(( null_pos + 1 )) 2>/dev/null > "$tmpbody"
    rm -f "$tmpraw"

    # Parse commit fields
    tree=$(grep '^tree '      "$tmpbody" | head -1 | cut -d' ' -f2 | tr -d '\n')
    parent=$(grep '^parent '  "$tmpbody" | head -1 | cut -d' ' -f2 | tr -d '\n')
    author=$(grep '^author '  "$tmpbody" | head -1 | sed 's/^author //')
    # Message: lines after the first blank line
    msg=$(awk 'found{print} /^$/{found=1}' "$tmpbody" | sed '/^$/d')
    rm -f "$tmpbody"

    # Format author name+email and date
    # author line: "Name <email> TIMESTAMP TIMEZONE"
    author_name=$(printf '%s' "$author" | sed 's/ <.*//')
    timestamp=$(printf '%s' "$author"  | grep -oE '[0-9]{9,10}' | tail -1)
    date_str=$(date -d "@$timestamp" '+%a %b %d %H:%M:%S %Y %z' 2>/dev/null \
               || date -r "$timestamp" '+%a %b %d %H:%M:%S %Y %z' 2>/dev/null \
               || printf '%s' "$timestamp")

    if [ "$ONELINE" = true ]; then
        first_line=$(printf '%s' "$msg" | head -1)
        printf '%s %s\n' "${current_sha:0:7}" "$first_line"
    else
        printf 'commit %s\n' "$current_sha"
        # Show merge info if multiple parents
        merge_parents=$(grep '^parent ' "$tmpbody" 2>/dev/null | wc -l | tr -d ' ')
        [ "$merge_parents" -gt 1 ] && \
            grep '^parent ' "$tmpbody" | awk '{printf "Merge: %s\n", substr($2,1,7)}'
        printf 'Author: %s\n' "$author_name"
        printf 'Date:   %s\n\n' "$date_str"
        printf '%s\n' "$msg" | sed 's/^/    /'
        printf '\n'
    fi

    current_sha="$parent"
    count=$(( count + 1 ))
done
```

### 4. Options

- `--oneline`: set `ONELINE=true`
- `-n N` / `-<N>`: set `MAX_COMMITS=N`
- `<branch>`: resolve the named branch ref instead of HEAD
- `--all`: loop over all refs in `.git/refs/heads/` and merge/deduplicate (advanced)
