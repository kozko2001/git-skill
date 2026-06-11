# Agent: git revert

Create a new commit that undoes the changes of a specific commit. Adds a new "undo" commit on top — does not delete history.

No `git` binary, no `python3`. Uses `gzip`/`dd`/`awk` for object access, then delegates commit creation to `agents/commit.md` logic.

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
            printf '%s %s %s\n' "$mode" "$sha" "$path"
        fi
    done < <(parse_tree_body "$tmptree")
    rm -f "$tmptree"
}
```

---

## Steps

### 1. Identify the target commit

If the user gave a short SHA, resolve it:

```bash
SHORT="REPLACE_SHORT_SHA"    # e.g. "a1b2c3d"
TARGET_SHA=$(find .git/objects -type f | while IFS= read -r f; do
    full="${f#.git/objects/}"
    full="${full/\//}"    # remove the slash between 2-char prefix and rest
    printf '%s\n' "$full"
done | grep "^$SHORT" | head -1)
[ -z "$TARGET_SHA" ] && echo "error: object '$SHORT' not found" && exit 1
```

### 2. Read the target commit

```bash
tmpraw=$(mktemp); tmpbody=$(mktemp)
git_decompress ".git/objects/${TARGET_SHA:0:2}/${TARGET_SHA:2}" > "$tmpraw"
obj_body_to_file "$tmpraw" "$tmpbody"; rm -f "$tmpraw"

target_tree=$(grep '^tree '   "$tmpbody" | awk '{print $2}')
parent_sha=$(grep  '^parent ' "$tmpbody" | head -1 | awk '{print $2}')
orig_msg=$(awk 'found{print} /^$/{found=1}' "$tmpbody" | head -1)
rm -f "$tmpbody"

[ -z "$parent_sha" ] && {
    echo "error: cannot revert the root commit (no parent)"
    echo "This would require deleting all files. Confirm with user before proceeding."
    exit 1
}
```

### 3. Read the parent commit's tree

```bash
tmpraw=$(mktemp); tmpbody=$(mktemp)
git_decompress ".git/objects/${parent_sha:0:2}/${parent_sha:2}" > "$tmpraw"
obj_body_to_file "$tmpraw" "$tmpbody"; rm -f "$tmpraw"
parent_tree=$(grep '^tree ' "$tmpbody" | awk '{print $2}')
rm -f "$tmpbody"
```

### 4. Compute the diff: target tree vs parent tree

```bash
tmptarget=$(mktemp); tmpparent=$(mktemp)
walk_tree "$target_tree" "" | sort > "$tmptarget"   # mode sha path
walk_tree "$parent_tree"  "" | sort > "$tmpparent"

# Files added in target (not in parent) → must be removed
added=$(comm -23 <(awk '{print $3}' "$tmptarget" | sort) \
                  <(awk '{print $3}' "$tmpparent" | sort))

# Files deleted in target (in parent, not in target) → must be restored
deleted=$(comm -13 <(awk '{print $3}' "$tmptarget" | sort) \
                    <(awk '{print $3}' "$tmpparent" | sort))

# Files modified in target → must be restored to parent version
modified=$(comm -12 <(awk '{print $3}' "$tmptarget" | sort) \
                     <(awk '{print $3}' "$tmpparent" | sort) | \
    while IFS= read -r path; do
        sha_t=$(grep " $path$" "$tmptarget" | awk '{print $2}')
        sha_p=$(grep " $path$" "$tmpparent" | awk '{print $2}')
        [ "$sha_t" != "$sha_p" ] && printf '%s %s\n' "$path" "$sha_p"
    done)
```

### 5. Apply the inverse changes to the working directory

**Remove added files** (they were added in the target commit → we remove them):
```bash
while IFS= read -r path; do
    [ -z "$path" ] && continue
    rm -f "$path"
    printf 'removed: %s\n' "$path"
done <<< "$added"
```

**Restore deleted and modified files** (from parent blob):
```bash
restore_blob() {
    local sha="$1" path="$2"
    local dir; dir=$(dirname "$path")
    [ "$dir" != "." ] && mkdir -p "$dir"
    local tmpraw=$(mktemp) tmpbody=$(mktemp)
    git_decompress ".git/objects/${sha:0:2}/${sha:2}" > "$tmpraw"
    obj_body_to_file "$tmpraw" "$tmpbody"; rm -f "$tmpraw"
    cp "$tmpbody" "$path"; rm -f "$tmpbody"
    printf 'restored: %s\n' "$path"
}

# Deleted files → restore
while IFS= read -r path; do
    [ -z "$path" ] && continue
    parent_sha_for_path=$(grep " $path$" "$tmpparent" | awk '{print $2}')
    restore_blob "$parent_sha_for_path" "$path"
done <<< "$deleted"

# Modified files → restore to parent version
while IFS=' ' read -r path parent_blob_sha; do
    [ -z "$path" ] && continue
    restore_blob "$parent_blob_sha" "$path"
done <<< "$modified"

rm -f "$tmptarget" "$tmpparent"
```

### 6. Create the revert commit

Use the commit agent logic (`agents/commit.md`) with this message:

```bash
COMMIT_MESSAGE=$(printf 'Revert "%s"\n\nThis reverts commit %s.' "$orig_msg" "$TARGET_SHA")
```

Run the full commit flow (blob → tree → commit → update ref) from `agents/commit.md`.

### 7. Report

```
[main abc1234] Revert "original commit message"
 2 file(s) changed
```
