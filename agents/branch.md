# Agent: git branch / checkout

List, create, or switch branches. No `git` binary.

Read `references/git-internals.md` first for `git_decompress`, `first_null_pos`, `parse_tree_body`.

## Inline helpers (paste at top of every bash block)

```bash
git_decompress() {
    local obj="$1"; local size; size=$(wc -c < "$obj"); local ds=$(( size - 6 ))
    { printf '\x1f\x8b\x08\x00\x00\x00\x00\x00\x00\x03'
      dd if="$obj" bs=1 skip=2 count="$ds" 2>/dev/null
      printf '\x00\x00\x00\x00\x00\x00\x00\x00'
    } | gzip -d -f -q 2>/dev/null
}

first_null_pos() {
    od -A n -t u1 -v "$1" | tr -s ' \n' '\n' | grep -v '^$' | \
    awk 'NR>0{ if($1+0==0){print NR-1; exit} }'
}

obj_body_to_file() {            # $1=rawfile $2=outfile
    local pos; pos=$(first_null_pos "$1")
    dd if="$1" bs=1 skip=$(( pos + 1 )) 2>/dev/null > "$2"
}

parse_tree_body() {
    local body="$1"
    od -A n -t u1 -v "$body" | tr -s ' \n' ' ' | sed 's/^ //;s/ $//' | \
    awk '{ n=split($0,b," "); i=1
           while(i<=n){ mode=""; while(i<=n&&b[i]+0!=32){mode=mode sprintf("%c",b[i]+0);i++}; i++
             name=""; while(i<=n&&b[i]+0!=0){name=name sprintf("%c",b[i]+0);i++}; i++
             sha=""; for(j=0;j<20&&i<=n;j++){sha=sha sprintf("%02x",b[i]+0);i++}
             if(mode!="") print mode, sha, name } }'
}
```

---

## List branches

```bash
current=$(cat .git/HEAD | sed 's|ref: refs/heads/||')
for branch in .git/refs/heads/*; do
    name=$(basename "$branch")
    if [ "$name" = "$current" ]; then
        printf '* %s\n' "$name"
    else
        printf '  %s\n' "$name"
    fi
done
```

---

## Create a new branch (without switching)

```bash
BRANCH_NAME="REPLACE"

# Get current HEAD SHA
head=$(cat .git/HEAD)
case "$head" in
    ref:\ *) sha=$(cat ".git/${head#ref: }" 2>/dev/null | tr -d '\n') ;;
    *)       sha=$(printf '%s' "$head" | tr -d '\n') ;;
esac

[ -z "$sha" ] && echo "Error: no commits yet — cannot create a branch" && exit 1

printf '%s\n' "$sha" > ".git/refs/heads/$BRANCH_NAME"
printf "Branch '%s' created at %s\n" "$BRANCH_NAME" "${sha:0:7}"
```

---

## Switch to an existing branch

```bash
BRANCH="REPLACE"
ref_file=".git/refs/heads/$BRANCH"
[ ! -f "$ref_file" ] && echo "error: branch '$BRANCH' does not exist" && exit 1

commit_sha=$(cat "$ref_file" | tr -d '\n')

# ── Step 1: get tree SHA from commit ──────────────────────────────────────────
tmpraw=$(mktemp)
git_decompress ".git/objects/${commit_sha:0:2}/${commit_sha:2}" > "$tmpraw"
tmpbody=$(mktemp)
obj_body_to_file "$tmpraw" "$tmpbody"
tree_sha=$(grep '^tree ' "$tmpbody" | head -1 | awk '{print $2}' | tr -d '\n')
rm -f "$tmpraw" "$tmpbody"

# ── Step 2: walk tree recursively, collect all (path → blob_sha) ──────────────
declare -A file_map   # bash 4+ associative array

walk_tree() {
    local tree_sha="$1"
    local prefix="$2"
    local tmpraw tmpbody tmptree

    tmpraw=$(mktemp)
    git_decompress ".git/objects/${tree_sha:0:2}/${tree_sha:2}" > "$tmpraw"
    tmptree=$(mktemp)
    obj_body_to_file "$tmpraw" "$tmptree"
    rm -f "$tmpraw"

    while IFS=' ' read -r mode sha name; do
        local path="${prefix:+$prefix/}$name"
        if [ "$mode" = "40000" ] || [ "$mode" = "040000" ]; then
            walk_tree "$sha" "$path"
        else
            file_map["$path"]="$sha:$mode"
        fi
    done < <(parse_tree_body "$tmptree")
    rm -f "$tmptree"
}

walk_tree "$tree_sha" ""

# ── Step 3: write each file to disk ──────────────────────────────────────────
for path in "${!file_map[@]}"; do
    entry="${file_map[$path]}"
    blob_sha="${entry%%:*}"
    mode="${entry##*:}"

    dir=$(dirname "$path")
    [ "$dir" != "." ] && mkdir -p "$dir"

    tmpraw=$(mktemp)
    git_decompress ".git/objects/${blob_sha:0:2}/${blob_sha:2}" > "$tmpraw"
    tmpbody=$(mktemp)
    obj_body_to_file "$tmpraw" "$tmpbody"
    cp "$tmpbody" "$path"
    rm -f "$tmpraw" "$tmpbody"

    [ "$mode" = "100755" ] && chmod 755 "$path"
done

# ── Step 4: update HEAD ───────────────────────────────────────────────────────
printf 'ref: refs/heads/%s\n' "$BRANCH" > .git/HEAD
printf "Switched to branch '%s'\n" "$BRANCH"
```

---

## Create and switch (checkout -b)

1. Create the branch (point it at current HEAD SHA — see "Create" above)
2. Write HEAD:

```bash
printf 'ref: refs/heads/%s\n' "$BRANCH_NAME" > .git/HEAD
printf "Switched to a new branch '%s'\n" "$BRANCH_NAME"
```

No working directory changes needed — files are already correct.

---

## Delete a branch

```bash
BRANCH="REPLACE"
current=$(cat .git/HEAD | sed 's|ref: refs/heads/||' | tr -d '\n')
[ "$current" = "$BRANCH" ] && echo "error: cannot delete the branch you are on" && exit 1
rm -f ".git/refs/heads/$BRANCH"
printf "Deleted branch '%s'\n" "$BRANCH"
```
