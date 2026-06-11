# Agent: git push

Push local commits to a remote. Two transport methods — pick based on remote URL and available credentials.

No `git` binary, no `python3`. Uses `curl`+`jq` (API) or `ssh`+`gzip`+`dd` (SSH git protocol).

## Step 0: Read repo state

```bash
head=$(cat .git/HEAD)
branch=$(printf '%s' "$head" | sed 's|ref: refs/heads/||' | tr -d '\n')
local_sha=$(cat ".git/refs/heads/$branch" 2>/dev/null | tr -d '\n')
[ -z "$local_sha" ] && echo "Nothing to push (no commits)" && exit 0

remote_url=$(awk '/^\[remote "origin"\]/{f=1} f&&/url *=/{gsub(/.*= */,"");print;exit}' .git/config)
owner_repo=$(printf '%s' "$remote_url" | sed -E 's|git@github\.com:||;s|https://github\.com/||;s|\.git$||')
owner=$(printf '%s' "$owner_repo" | cut -d/ -f1)
repo=$(printf '%s' "$owner_repo" | cut -d/ -f2)
```

## Step 1: Choose transport

- Remote URL starts with `git@` → use **Method B (SSH)**
- `$GITHUB_TOKEN` set → use **Method A (GitHub API)**
- Otherwise: ask user for a token

---

## Method A: GitHub API

### A1. Get remote HEAD

```bash
remote_resp=$(curl -sf -H "Authorization: Bearer $GITHUB_TOKEN" \
    "https://api.github.com/repos/$owner/$repo/git/refs/heads/$branch")
remote_sha=$(printf '%s' "$remote_resp" | jq -r '.object.sha // empty')
# If 404 (branch doesn't exist on remote), remote_sha will be empty
```

### A2. Find new commits to push

Walk back from `$local_sha` until we reach `$remote_sha` (or no more parents).
Objects are stored body-only (no `type size\0` prefix) — decompress gives the body directly.

```bash
git_decompress() {
    local obj="$1"; local size; size=$(wc -c < "$obj"); local ds=$(( size - 6 ))
    { printf '\x1f\x8b\x08\x00\x00\x00\x00\x00\x00\x03'
      dd if="$obj" bs=1 skip=2 count="$ds" 2>/dev/null
      printf '\x00\x00\x00\x00\x00\x00\x00\x00'
    } | gzip -d -f -q 2>/dev/null
}

to_push=()
sha="$local_sha"
while [ -n "$sha" ] && [ "$sha" != "$remote_sha" ]; do
    to_push=("$sha" "${to_push[@]}")   # prepend → oldest first
    tmpbody=$(mktemp)
    git_decompress ".git/objects/${sha:0:2}/${sha:2}" > "$tmpbody"
    sha=$(grep '^parent ' "$tmpbody" | head -1 | awk '{print $2}')
    rm -f "$tmpbody"
done
```

### A3. Upload each commit (oldest first)

```bash
parse_tree_body() {
    od -A n -t u1 -v "$1" | tr -s ' \n' ' ' | sed 's/^ //;s/ $//' | \
    awk '{n=split($0,b," ");i=1;while(i<=n){mode="";while(i<=n&&b[i]+0!=32){mode=mode sprintf("%c",b[i]+0);i++};i++;name="";while(i<=n&&b[i]+0!=0){name=name sprintf("%c",b[i]+0);i++};i++;sha="";for(j=0;j<20&&i<=n;j++){sha=sha sprintf("%02x",b[i]+0);i++};if(mode!="")print mode,sha,name}}'
}

walk_tree() {
    local tree_sha="$1" prefix="$2"
    local tmptree; tmptree=$(mktemp)
    git_decompress ".git/objects/${tree_sha:0:2}/${tree_sha:2}" > "$tmptree"
    while IFS=' ' read -r mode sha name; do
        local path="${prefix:+$prefix/}$name"
        if [ "$mode" = "40000" ] || [ "$mode" = "040000" ]; then walk_tree "$sha" "$path"
        else printf '%s %s\n' "$sha" "$path"; fi
    done < <(parse_tree_body "$tmptree")
    rm -f "$tmptree"
}

API="https://api.github.com/repos/$owner/$repo/git"
prev_remote_sha="$remote_sha"

for commit_sha in "${to_push[@]}"; do
    tmpbody=$(mktemp)
    git_decompress ".git/objects/${commit_sha:0:2}/${commit_sha:2}" > "$tmpbody"
    tree_sha=$(grep '^tree '  "$tmpbody" | awk '{print $2}')
    author=$(grep  '^author ' "$tmpbody" | sed 's/^author //')
    msg=$(awk 'f{print} /^$/{f=1}' "$tmpbody" | head -1)
    rm -f "$tmpbody"

    # Upload blobs and build tree entries for GitHub API
    gh_tree_entries="[]"
    while IFS=' ' read -r blob_sha path; do
        tmpbody=$(mktemp)
        git_decompress ".git/objects/${blob_sha:0:2}/${blob_sha:2}" > "$tmpbody"
        encoded=$(base64 < "$tmpbody"); rm -f "$tmpbody"
        gh_blob_sha=$(curl -sf -X POST \
            -H "Authorization: Bearer $GITHUB_TOKEN" \
            -H "Content-Type: application/json" \
            "$API/blobs" \
            -d "{\"content\":\"$encoded\",\"encoding\":\"base64\"}" | jq -r '.sha')
        gh_tree_entries=$(printf '%s' "$gh_tree_entries" | \
            jq --arg p "$path" --arg m "100644" --arg s "$gh_blob_sha" \
            '. + [{"path":$p,"mode":$m,"type":"blob","sha":$s}]')
    done < <(walk_tree "$tree_sha" "")

    # Create tree
    tree_payload=$(jq -n --argjson t "$gh_tree_entries" '{"tree":$t}')
    [ -n "$prev_remote_sha" ] && \
        tree_payload=$(printf '%s' "$tree_payload" | jq --arg b "$prev_remote_sha" '. + {"base_tree":$b}')
    gh_tree_sha=$(curl -sf -X POST \
        -H "Authorization: Bearer $GITHUB_TOKEN" \
        -H "Content-Type: application/json" \
        "$API/trees" -d "$tree_payload" | jq -r '.sha')

    # Author date
    ts=$(printf '%s' "$author" | grep -oE '[0-9]{9,10}' | tail -1)
    date_iso=$(date -d "@$ts" -u '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -r "$ts" -u '+%Y-%m-%dT%H:%M:%SZ')
    author_name=$(printf '%s' "$author" | sed 's/ <.*//')
    author_email=$(printf '%s' "$author" | grep -oP '(?<=<)[^>]+')

    # Create commit
    commit_payload=$(jq -n \
        --arg msg "$msg" --arg tree "$gh_tree_sha" \
        --arg name "$author_name" --arg email "$author_email" --arg date "$date_iso" \
        '{"message":$msg,"tree":$tree,"author":{"name":$name,"email":$email,"date":$date}}')
    [ -n "$prev_remote_sha" ] && \
        commit_payload=$(printf '%s' "$commit_payload" | jq --arg p "$prev_remote_sha" '. + {"parents":[$p]}')
    prev_remote_sha=$(curl -sf -X POST \
        -H "Authorization: Bearer $GITHUB_TOKEN" \
        -H "Content-Type: application/json" \
        "$API/commits" -d "$commit_payload" | jq -r '.sha')

    printf 'Pushed: %s %s\n' "${prev_remote_sha:0:7}" "$msg"
done

# Update or create remote ref
resp=$(curl -sf -X PATCH \
    -H "Authorization: Bearer $GITHUB_TOKEN" \
    -H "Content-Type: application/json" \
    "$API/refs/heads/$branch" \
    -d "{\"sha\":\"$prev_remote_sha\",\"force\":false}" 2>/dev/null)
if printf '%s' "$resp" | jq -e '.ref' > /dev/null 2>&1; then
    printf 'Updated refs/heads/%s -> %s\n' "$branch" "${prev_remote_sha:0:7}"
else
    curl -sf -X POST \
        -H "Authorization: Bearer $GITHUB_TOKEN" \
        -H "Content-Type: application/json" \
        "$API/refs" \
        -d "{\"ref\":\"refs/heads/$branch\",\"sha\":\"$prev_remote_sha\"}"
    printf 'Created refs/heads/%s -> %s\n' "$branch" "${prev_remote_sha:0:7}"
fi

# Store remote tracking ref
mkdir -p ".git/refs/remotes/origin"
printf '%s\n' "$prev_remote_sha" > ".git/refs/remotes/origin/$branch"
```

---

## Method B: SSH git protocol

Uses `ssh` binary + pack file built with `gzip`/`dd`/`sha1sum`. No `git` binary.

### B1. Parse SSH remote

```bash
# From remote_url: "git@github.com:owner/repo.git"
ssh_host="git@github.com"
repo_path="${remote_url#*:}"     # "owner/repo.git"
```

### B2. Collect objects and build the PACK file

Objects are stored body-only (no `type size\0` prefix). Track types during BFS traversal
instead of reading them from file headers.

```bash
git_decompress() {
    local obj=".git/objects/${1:0:2}/${1:2}"
    local size; size=$(wc -c < "$obj"); local ds=$(( size - 6 ))
    { printf '\x1f\x8b\x08\x00\x00\x00\x00\x00\x00\x03'
      dd if="$obj" bs=1 skip=2 count="$ds" 2>/dev/null
      printf '\x00\x00\x00\x00\x00\x00\x00\x00'
    } | gzip -d -f -q 2>/dev/null
}

parse_tree_body() {
    od -A n -t u1 -v "$1" | tr -s ' \n' ' ' | sed 's/^ //;s/ $//' | \
    awk '{n=split($0,b," ");i=1;while(i<=n){mode="";while(i<=n&&b[i]+0!=32){mode=mode sprintf("%c",b[i]+0);i++};i++;name="";while(i<=n&&b[i]+0!=0){name=name sprintf("%c",b[i]+0);i++};i++;sha="";for(j=0;j<20&&i<=n;j++){sha=sha sprintf("%02x",b[i]+0);i++};if(mode!="")print mode,sha,name}}'
}

# Get remote SHA (empty = new branch)
remote_sha=$(cat ".git/refs/remotes/origin/$branch" 2>/dev/null | tr -d '\n')

# BFS: collect (sha, type) pairs not already on remote
objects_tmp=$(mktemp)

commits=("$local_sha")
while [ "${#commits[@]}" -gt 0 ]; do
    sha="${commits[0]}"; commits=("${commits[@]:1}")
    grep -qF "$sha" "$objects_tmp" && continue
    [ "$sha" = "$remote_sha" ] && continue
    printf '%s commit\n' "$sha" >> "$objects_tmp"
    tmpbody=$(mktemp)
    git_decompress "$sha" > "$tmpbody"
    tree_sha=$(grep '^tree ' "$tmpbody" | awk '{print $2}')
    grep -qF "$tree_sha" "$objects_tmp" || printf '%s tree\n' "$tree_sha" >> "$objects_tmp"
    while IFS= read -r p; do commits+=("$p"); done \
        < <(grep '^parent ' "$tmpbody" | awk '{print $2}')
    rm -f "$tmpbody"
done

pending=()
while IFS=' ' read -r sha type; do
    [ "$type" = "tree" ] && pending+=("$sha")
done < "$objects_tmp"

while [ "${#pending[@]}" -gt 0 ]; do
    sha="${pending[0]}"; pending=("${pending[@]:1}")
    tmpbody=$(mktemp)
    git_decompress "$sha" > "$tmpbody"
    while IFS=' ' read -r mode child_sha name; do
        grep -qF "$child_sha" "$objects_tmp" && continue
        if [ "$mode" = "40000" ] || [ "$mode" = "040000" ]; then
            printf '%s tree\n' "$child_sha" >> "$objects_tmp"
            pending+=("$child_sha")
        else
            printf '%s blob\n' "$child_sha" >> "$objects_tmp"
        fi
    done < <(parse_tree_body "$tmpbody")
    rm -f "$tmpbody"
done

obj_count=$(wc -l < "$objects_tmp")

# Build pack file: PACK header + objects (varlen header + zlib body) + SHA checksum
pack_tmp=$(mktemp)
{
    printf 'PACK'
    printf '\x00\x00\x00\x02'    # version 2
    printf "$(printf '%08x' "$obj_count" | sed 's/../\\x&/g')"

    while IFS=' ' read -r sha type_str; do
        [ -z "$sha" ] && continue
        case "$type_str" in
            commit) type_num=1 ;; tree) type_num=2 ;; blob) type_num=3 ;; *) continue ;;
        esac

        # Decompress body (stored body-only)
        tmpbody=$(mktemp)
        git_decompress "$sha" > "$tmpbody"
        body_size=$(wc -c < "$tmpbody")

        # Variable-length size/type header (git pack format)
        size="$body_size"
        first=$(( (type_num << 4) | (size & 0xF) ))
        size=$(( size >> 4 ))
        header_hex=""
        while [ "$size" -gt 0 ]; do
            header_hex="${header_hex}$(printf '%02x' $(( first | 0x80 )))"
            first=$(( size & 0x7F ))
            size=$(( size >> 7 ))
        done
        header_hex="${header_hex}$(printf '%02x' "$first")"
        printf "$(printf '%s' "$header_hex" | sed 's/../\\x&/g')"

        # zlib-compress body (proper zlib: \x78\x9c header + deflate stream + Adler-32)
        tmpgz=$(mktemp)
        gzip -1 -c -n "$tmpbody" > "$tmpgz"
        gz_size=$(wc -c < "$tmpgz")
        ds=$(( gz_size - 18 ))
        adler=$(od -A n -t u1 -v "$tmpbody" | tr -s ' \n' ' ' | sed 's/^ //;s/ $//' | \
            awk 'BEGIN{a=1;b=0;m=65521}
                 {n=split($0,v," ");for(i=1;i<=n;i++){d=v[i]+0;a=(a+d)%m;b=(b+a)%m}}
                 END{printf "%02x%02x%02x%02x",int(b/256)%256,b%256,int(a/256)%256,a%256}')
        { printf '\x78\x9c'
          dd if="$tmpgz" bs=1 skip=10 count="$ds" 2>/dev/null
          printf "$(printf '%s' "$adler" | sed 's/../\\x&/g')"
        }
        rm -f "$tmpbody" "$tmpgz"
    done < "$objects_tmp"
} > "$pack_tmp"

pack_sha=$(sha1sum "$pack_tmp" | cut -d' ' -f1)
printf "$(printf '%s' "$pack_sha" | sed 's/../\\x&/g')" >> "$pack_tmp"
rm -f "$objects_tmp"
```

### B3. Send via SSH git protocol

```bash
zero_sha='0000000000000000000000000000000000000000'
old_sha="${remote_sha:-$zero_sha}"

# Build pkt-line: "old new ref\0capabilities\n"
# NUL byte separates ref update from capabilities; length must include it.
ref_update="${old_sha} ${local_sha} refs/heads/${branch}"
capabilities="report-status side-band-64k agent=git/2.0"
payload_len=$(( ${#ref_update} + 1 + ${#capabilities} + 1 ))
pkt_len=$(( payload_len + 4 ))

{
    printf '%04x' "$pkt_len"
    printf '%s' "$ref_update"
    printf '\0'
    printf '%s\n' "$capabilities"
    printf '0000'
    cat "$pack_tmp"
} | ssh -T "$ssh_host" "git-receive-pack '$repo_path'" 2>&1

rm -f "$pack_tmp"
printf '\nTo %s\n   %s..%s  %s -> %s\n' \
    "$remote_url" "${old_sha:0:7}" "${local_sha:0:7}" "$branch" "$branch"

mkdir -p ".git/refs/remotes/origin"
printf '%s\n' "$local_sha" > ".git/refs/remotes/origin/$branch"
```
