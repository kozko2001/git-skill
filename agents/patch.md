# Agent: git patch (format-patch / diff)

Create a unified diff patch file. Uses the system `diff -u` binary — no python, no git binary.

Read `references/git-internals.md` for `git_decompress`, `first_null_pos`, `parse_tree_body`.

## Inline helpers

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
    awk 'NR>0{if($1+0==0){print NR-1;exit}}'
}

obj_body_to_file() {
    local pos; pos=$(first_null_pos "$1")
    dd if="$1" bs=1 skip=$(( pos + 1 )) 2>/dev/null > "$2"
}

parse_tree_body() {
    od -A n -t u1 -v "$1" | tr -s ' \n' ' ' | sed 's/^ //;s/ $//' | \
    awk '{n=split($0,b," ");i=1;while(i<=n){mode="";while(i<=n&&b[i]+0!=32){mode=mode sprintf("%c",b[i]+0);i++};i++;name="";while(i<=n&&b[i]+0!=0){name=name sprintf("%c",b[i]+0);i++};i++;sha="";for(j=0;j<20&&i<=n;j++){sha=sha sprintf("%02x",b[i]+0);i++};if(mode!="")print mode,sha,name}}'
}

# Walk a tree recursively; output: "blob_sha relative/path"
walk_tree() {
    local tree_sha="$1" prefix="$2"
    local tmpraw=$(mktemp) tmptree=$(mktemp)
    git_decompress ".git/objects/${tree_sha:0:2}/${tree_sha:2}" > "$tmpraw"
    obj_body_to_file "$tmpraw" "$tmptree"; rm -f "$tmpraw"
    while IFS=' ' read -r mode sha name; do
        local path="${prefix:+$prefix/}$name"
        if [ "$mode" = "40000" ] || [ "$mode" = "040000" ]; then
            walk_tree "$sha" "$path"
        else
            printf '%s %s\n' "$sha" "$path"
        fi
    done < <(parse_tree_body "$tmptree")
    rm -f "$tmptree"
}
```

---

## Case 1: Diff two commits

```bash
SHA_A="REPLACE_OLDER_SHA"    # e.g. parent commit
SHA_B="REPLACE_NEWER_SHA"    # e.g. HEAD

OUTPUT_PATCH="changes.patch"

# Get tree SHAs
get_tree_sha() {
    local sha="$1"
    local tmpraw=$(mktemp) tmpbody=$(mktemp)
    git_decompress ".git/objects/${sha:0:2}/${sha:2}" > "$tmpraw"
    obj_body_to_file "$tmpraw" "$tmpbody"; rm -f "$tmpraw"
    grep '^tree ' "$tmpbody" | awk '{print $2}'; rm -f "$tmpbody"
}

tree_a=$(get_tree_sha "$SHA_A")
tree_b=$(get_tree_sha "$SHA_B")

# Build path→sha maps (stored as files)
tmpmap_a=$(mktemp); tmpmap_b=$(mktemp)
walk_tree "$tree_a" "" | sort > "$tmpmap_a"
walk_tree "$tree_b" "" | sort > "$tmpmap_b"

# Collect all paths
all_paths=$(cat "$tmpmap_a" "$tmpmap_b" | awk '{print $2}' | sort -u)

> "$OUTPUT_PATCH"

while IFS= read -r path; do
    sha_a=$(grep " $path$" "$tmpmap_a" | awk '{print $1}')
    sha_b=$(grep " $path$" "$tmpmap_b" | awk '{print $1}')

    [ "$sha_a" = "$sha_b" ] && continue   # identical — skip

    # Extract blob contents to tmpfiles
    tmpa=$(mktemp); tmpb=$(mktemp)

    if [ -n "$sha_a" ]; then
        tmpraw=$(mktemp)
        git_decompress ".git/objects/${sha_a:0:2}/${sha_a:2}" > "$tmpraw"
        obj_body_to_file "$tmpraw" "$tmpa"; rm -f "$tmpraw"
    fi

    if [ -n "$sha_b" ]; then
        tmpraw=$(mktemp)
        git_decompress ".git/objects/${sha_b:0:2}/${sha_b:2}" > "$tmpraw"
        obj_body_to_file "$tmpraw" "$tmpb"; rm -f "$tmpraw"
    fi

    # Generate unified diff (diff exits 1 when files differ — that's normal)
    diff -u \
        --label "a/$path" \
        --label "b/$path" \
        "$tmpa" "$tmpb" >> "$OUTPUT_PATCH" || true

    rm -f "$tmpa" "$tmpb"
done <<< "$all_paths"

rm -f "$tmpmap_a" "$tmpmap_b"
printf 'Patch written to %s\n' "$OUTPUT_PATCH"
```

---

## Case 2: Single commit vs its parent

```bash
COMMIT_SHA="REPLACE"

# Get parent SHA
tmpraw=$(mktemp); tmpbody=$(mktemp)
git_decompress ".git/objects/${COMMIT_SHA:0:2}/${COMMIT_SHA:2}" > "$tmpraw"
obj_body_to_file "$tmpraw" "$tmpbody"; rm -f "$tmpraw"

parent_sha=$(grep '^parent ' "$tmpbody" | head -1 | awk '{print $2}')
author=$(grep '^author ' "$tmpbody" | sed 's/^author //')
msg=$(awk 'found{print} /^$/{found=1}' "$tmpbody" | head -1)
rm -f "$tmpbody"

# Optionally prepend email-format header
printf 'From %s Mon Sep 17 00:00:00 2001\nFrom: %s\nSubject: [PATCH] %s\n\n---\n' \
    "$COMMIT_SHA" "$author" "$msg" > changes.patch

# Then diff parent → commit and append
SHA_A="$parent_sha"
SHA_B="$COMMIT_SHA"
# ... run Case 1 code above, appending to changes.patch
```

---

## Case 3: HEAD vs working tree

```bash
OUTPUT_PATCH="working.patch"

# Resolve HEAD tree
head=$(cat .git/HEAD)
case "$head" in
    ref:\ *) commit_sha=$(cat ".git/${head#ref: }" | tr -d '\n') ;;
    *)       commit_sha="$head" ;;
esac

tmpraw=$(mktemp); tmpbody=$(mktemp)
git_decompress ".git/objects/${commit_sha:0:2}/${commit_sha:2}" > "$tmpraw"
obj_body_to_file "$tmpraw" "$tmpbody"; rm -f "$tmpraw"
head_tree=$(grep '^tree ' "$tmpbody" | awk '{print $2}'); rm -f "$tmpbody"

tmpmap=$(mktemp)
walk_tree "$head_tree" "" | sort > "$tmpmap"

> "$OUTPUT_PATCH"

# Compare each HEAD blob vs its on-disk counterpart
while IFS=' ' read -r sha path; do
    [ ! -f "$path" ] && continue   # deleted — show as empty vs disk
    tmpblob=$(mktemp); tmpraw=$(mktemp)
    git_decompress ".git/objects/${sha:0:2}/${sha:2}" > "$tmpraw"
    obj_body_to_file "$tmpraw" "$tmpblob"; rm -f "$tmpraw"
    diff -u --label "a/$path" --label "b/$path" "$tmpblob" "$path" >> "$OUTPUT_PATCH" || true
    rm -f "$tmpblob"
done < "$tmpmap"

# New files (on disk but not in HEAD)
find . -not -path './.git/*' -type f | sort | while IFS= read -r path; do
    norm="${path#./}"
    grep -q " $norm$" "$tmpmap" && continue
    diff -u --label "a/$norm" --label "b/$norm" /dev/null "$path" >> "$OUTPUT_PATCH" || true
done

rm -f "$tmpmap"

if [ -s "$OUTPUT_PATCH" ]; then
    printf 'Patch written to %s\n' "$OUTPUT_PATCH"
else
    printf 'No changes detected\n'
fi
```
