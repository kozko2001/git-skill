# Agent: git commit

Create a new commit from the working tree. No `git` binary, no python3.

Read `references/git-internals.md` first — provides `git_compress`, `git_hash_object`, `git_decompress`.

## Before starting

Ask the user for (if not already provided):
1. **Commit message** — required
2. **Files to commit** — default: all tracked files; user can specify a subset
3. **Author** — read from `.git/config`; ask only if missing

```bash
# Read author from config
author_name=$(awk '/^\[user\]/{u=1} u&&/name *=/{gsub(/.*= */,""); print; exit}' .git/config)
author_email=$(awk '/^\[user\]/{u=1} u&&/email *=/{gsub(/.*= */,""); print; exit}' .git/config)
```

---

## Inline helpers

```bash
git_compress() {
    local infile="$1" outfile="$2"
    local tmpgz; tmpgz=$(mktemp)
    gzip -1 -c -n "$infile" > "$tmpgz"
    local gz_size; gz_size=$(wc -c < "$tmpgz")
    local ds=$(( gz_size - 18 ))
    local adler
    adler=$(od -A n -t u1 -v "$infile" | tr -s ' \n' ' ' | sed 's/^ //;s/ $//' | \
        awk 'BEGIN{a=1;b=0;m=65521}
             {n=split($0,v," ");for(i=1;i<=n;i++){d=v[i]+0;a=(a+d)%m;b=(b+a)%m}}
             END{printf "%02x%02x%02x%02x",int(b/256)%256,b%256,int(a/256)%256,a%256}')
    mkdir -p "$(dirname "$outfile")"
    { printf '\x78\x9c'
      dd if="$tmpgz" bs=1 skip=10 count="$ds" 2>/dev/null
      printf "$(printf '%s' "$adler" | sed 's/../\\x&/g')"
    } > "$outfile"
    rm -f "$tmpgz"
}

git_hash_object() {
    local type="$1" infile="$2"
    local size; size=$(wc -c < "$infile")
    { printf "%s %d\0" "$type" "$size"; cat "$infile"; } | sha1sum | cut -d' ' -f1
}
```

---

## Steps

### 1. Collect files

```bash
# All files except .git/
mapfile -t FILES < <(find . -not -path './.git/*' -not -name '.git' -type f | sort)
# Or a user-specified list:
# FILES=("src/main.c" "README.md")
```

If a `.gitignore` exists, filter out matching patterns:
```bash
while IFS= read -r pat; do
    [[ "$pat" =~ ^#|^$ ]] && continue
    FILES=( "${FILES[@]/$pat}" )     # basic substring filter; for globs use find -not -name
done < .gitignore 2>/dev/null
```

### 2. Create blob objects for all files

```bash
declare -A blob_shas   # path → sha

for f in "${FILES[@]}"; do
    [ -z "$f" ] && continue
    sha=$(git_hash_object blob "$f")
    obj_path=".git/objects/${sha:0:2}/${sha:2}"
    [ ! -f "$obj_path" ] && git_compress "$f" "$obj_path"
    blob_shas["$f"]="$sha"
done
```

### 3. Build tree objects (bottom-up)

Sort directories deepest-first so subtrees exist before their parents:

```bash
hex_to_bin() { printf "$(printf '%s' "$1" | sed 's/../\\x&/g')"; }

build_tree() {
    local dir="$1"   # e.g. "." or "src"
    local prefix="${dir#./}"
    [ "$prefix" = "." ] && prefix=""

    local tmpentries; tmpentries=$(mktemp)

    # Sub-directories
    while IFS= read -r subdir; do
        local name; name=$(basename "$subdir")
        local sub_sha; sub_sha=$(build_tree "$subdir")
        [ -z "$sub_sha" ] && continue
        printf '%s %s %s\n' "40000" "$name" "$sub_sha"
    done < <(find "$dir" -mindepth 1 -maxdepth 1 -type d ! -name '.git' | sort) >> "$tmpentries"

    # Files in this directory
    for f in "${FILES[@]}"; do
        [ -z "$f" ] && continue
        local fdir; fdir=$(dirname "$f")
        local fname; fname=$(basename "$f")
        # Normalize: strip leading ./
        local norm_dir="${fdir#./}"
        [ "$norm_dir" = "." ] && norm_dir=""
        [ "$norm_dir" != "$prefix" ] && continue

        local sha="${blob_shas[$f]}"
        [ -x "$f" ] && mode="100755" || mode="100644"
        printf '%s %s %s\n' "$mode" "$fname" "$sha"
    done >> "$tmpentries"

    # Sort entries (files by name; dirs by name + "/")
    local sorted; sorted=$(sort "$tmpentries")
    rm -f "$tmpentries"

    [ -z "$sorted" ] && return 0  # empty dir → no tree

    # Build binary tree content
    local tmptree; tmptree=$(mktemp)
    while IFS=' ' read -r mode name sha; do
        printf '%s %s\0' "$mode" "$name" >> "$tmptree"
        hex_to_bin "$sha"              >> "$tmptree"
    done <<< "$sorted"

    local tree_sha; tree_sha=$(git_hash_object tree "$tmptree")
    local obj_path=".git/objects/${tree_sha:0:2}/${tree_sha:2}"
    [ ! -f "$obj_path" ] && git_compress "$tmptree" "$obj_path"
    rm -f "$tmptree"

    printf '%s' "$tree_sha"
}

root_tree_sha=$(build_tree ".")
```

### 4. Resolve parent SHA

```bash
head=$(cat .git/HEAD)
case "$head" in
    ref:\ *) ref="${head#ref: }"; parent=$(cat ".git/$ref" 2>/dev/null | tr -d '\n') ;;
    *)       parent=$(printf '%s' "$head" | tr -d '\n') ;;
esac
branch=$(printf '%s' "$head" | sed 's|ref: refs/heads/||' | tr -d '\n')
```

### 5. Build and store the commit object

```bash
COMMIT_MESSAGE="REPLACE_WITH_ACTUAL_MESSAGE"
[ -z "$author_name" ]  && author_name="Unknown"
[ -z "$author_email" ] && author_email="unknown@example.com"

ts=$(date +%s)
tz=$(date +%z)
author_str="$author_name <$author_email> $ts $tz"

tmpcmt=$(mktemp)
{
    printf 'tree %s\n' "$root_tree_sha"
    [ -n "$parent" ] && printf 'parent %s\n' "$parent"
    printf 'author %s\ncommitter %s\n\n%s\n' "$author_str" "$author_str" "$COMMIT_MESSAGE"
} > "$tmpcmt"

commit_sha=$(git_hash_object commit "$tmpcmt")
obj_path=".git/objects/${commit_sha:0:2}/${commit_sha:2}"
git_compress "$tmpcmt" "$obj_path"
rm -f "$tmpcmt"
```

### 6. Update the branch ref

```bash
if printf '%s' "$head" | grep -q '^ref: '; then
    ref="${head#ref: }"
    mkdir -p "$(dirname ".git/$ref")"
    printf '%s\n' "$commit_sha" > ".git/$ref"
else
    printf '%s\n' "$commit_sha" > .git/HEAD
fi

printf '[%s %s] %s\n' "$branch" "${commit_sha:0:7}" "$COMMIT_MESSAGE"
printf '%d file(s) committed\n' "${#FILES[@]}"
```
