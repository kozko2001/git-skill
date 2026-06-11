# Git Internals Reference — POSIX Tools Only

No python3. No git binary. Every snippet below uses only: `gzip`, `gzip -d`, `dd`, `od`, `sha1sum`, `awk`, `sed`, `printf`, `find`, `wc`, `tr`, `cut`, `mktemp`.

---

## Object store layout

```
.git/
├── HEAD                        # "ref: refs/heads/main" or bare SHA (plain text)
├── config                      # INI format — plain text
├── refs/
│   ├── heads/<branch>          # plain text: 40-char commit SHA + newline
│   └── tags/<tag>
└── objects/
    ├── ab/                     # first 2 hex chars of SHA
    │   └── cdef1234...         # remaining 38 chars — zlib-compressed object
    └── pack/                   # (ignore for now)
```

---

## Object format (inside the zlib compression)

```
<type> <size>\x00<content>
```

| Type   | Content format |
|--------|----------------|
| blob   | raw file bytes |
| tree   | binary entries: `<mode> <name>\x00<20-byte-sha>` repeated, sorted by name |
| commit | plain text: `tree`, `parent`, `author`, `committer`, blank line, message |
| tag    | plain text |

Tree sort order: files by name, directories by name + "/" (so "foo" sorts after "foo.c" but "foo/" sorts after "foo").

---

## The zlib ↔ gzip header swap

**Why it works:** both zlib (RFC 1950) and gzip (RFC 1952) wrap the same deflate stream. Only the outer headers and checksums differ.

```
zlib:  [CMF FLG][deflate stream][Adler-32 4 bytes]
gzip:  [1f 8b 08 00 00 00 00 00 00 03][deflate stream][CRC32 4 bytes][ISIZE 4 bytes]
```

### Decompress a git object

```bash
git_decompress() {
    local obj="$1"              # path: .git/objects/ab/cdef...
    local size; size=$(wc -c < "$obj")
    local deflate_size=$(( size - 6 ))   # strip 2-byte zlib header + 4-byte adler32
    {
        printf '\x1f\x8b\x08\x00\x00\x00\x00\x00\x00\x03'  # gzip header (10 bytes)
        dd if="$obj" bs=1 skip=2 count="$deflate_size" 2>/dev/null
        printf '\x00\x00\x00\x00\x00\x00\x00\x00'           # fake CRC32 + ISIZE
    } | gzip -d -f -q 2>/dev/null
    # gzip flushes decompressed data to stdout before the CRC check.
    # The CRC mismatch error goes to stderr; 2>/dev/null discards it.
    # -f forces decompression; -q suppresses the warning.
}
```

### Compress content to zlib (for storing objects)

```bash
git_compress() {
    local infile="$1"
    local outfile="$2"
    local tmpgz; tmpgz=$(mktemp)

    gzip -1 -c -n "$infile" > "$tmpgz"          # -n: no filename/mtime (header stays 10 bytes)
    local gz_size; gz_size=$(wc -c < "$tmpgz")
    local deflate_size=$(( gz_size - 18 ))        # strip 10-byte header + 8-byte trailer

    # Adler-32 of original file (big-endian, 4 bytes)
    local adler
    adler=$(od -A n -t u1 -v "$infile" | tr -s ' \n' ' ' | sed 's/^ //;s/ $//' | \
        awk 'BEGIN{a=1;b=0;m=65521}
             { n=split($0,v," "); for(i=1;i<=n;i++){ d=v[i]+0; a=(a+d)%m; b=(b+a)%m } }
             END{ printf "%02x%02x%02x%02x", int(b/256)%256, b%256, int(a/256)%256, a%256 }')

    mkdir -p "$(dirname "$outfile")"
    {
        printf '\x78\x9c'                         # zlib header (valid: 0x789c % 31 == 0)
        dd if="$tmpgz" bs=1 skip=10 count="$deflate_size" 2>/dev/null
        printf "$(printf '%s' "$adler" | sed 's/../\\x&/g')"
    } > "$outfile"
    rm -f "$tmpgz"
}
```

---

## SHA-1 hashing

```bash
git_hash_object() {
    local type="$1"     # blob | tree | commit
    local infile="$2"
    local size; size=$(wc -c < "$infile")
    { printf "%s %d\0" "$type" "$size"; cat "$infile"; } | sha1sum | cut -d' ' -f1
}
```

The SHA of a stored object is computed over `"<type> <size>\x00<raw-content>"` — **not** over the compressed bytes.

---

## Reading an object's header and body

After decompressing, the raw bytes are `<header>\x00<body>`.

```bash
# Find byte offset of the first null byte
first_null_pos() {
    local file="$1"
    od -A n -t u1 -v "$file" | tr -s ' \n' '\n' | grep -v '^$' | \
    awk 'NR>0{ if($1+0==0){print NR-1; exit} }'
}

# Extract the header string (e.g. "commit 241")
obj_header() {
    local raw="$1"
    local pos; pos=$(first_null_pos "$raw")
    dd if="$raw" bs=1 count="$pos" 2>/dev/null
}

# Extract the body bytes (everything after the null byte)
obj_body() {
    local raw="$1"
    local pos; pos=$(first_null_pos "$raw")
    dd if="$raw" bs=1 skip=$(( pos + 1 )) 2>/dev/null
}
```

---

## Hex ↔ binary (no xxd)

```bash
# 40-char hex SHA → 20 raw binary bytes
hex_to_bin() {
    printf "$(printf '%s' "$1" | sed 's/../\\x&/g')"
}

# 20 raw bytes at byte-offset N in a file → 40-char hex string
bin_to_hex() {
    local file="$1" skip="$2"
    dd if="$file" bs=1 skip="$skip" count=20 2>/dev/null | od -A n -t x1 | tr -d ' \n'
}
```

---

## Parsing tree entries

Tree body is binary: `<mode-str> <name-str>\x00<20-byte-sha>` repeated.

```bash
# Outputs lines: "<mode> <sha-hex> <name>"
parse_tree_body() {
    local body_file="$1"
    od -A n -t u1 -v "$body_file" | tr -s ' \n' ' ' | sed 's/^ //;s/ $//' | \
    awk 'BEGIN{OFS=" "}
    {
        n = split($0, b, " ")
        i = 1
        while (i <= n) {
            # mode: read until space (32)
            mode = ""
            while (i <= n && b[i]+0 != 32) { mode = mode sprintf("%c", b[i]+0); i++ }
            i++  # skip space
            if (mode == "") break

            # name: read until null (0)
            name = ""
            while (i <= n && b[i]+0 != 0) { name = name sprintf("%c", b[i]+0); i++ }
            i++  # skip null

            # sha: next 20 bytes as hex
            sha = ""
            for (j = 0; j < 20 && i <= n; j++) { sha = sha sprintf("%02x", b[i]+0); i++ }

            print mode, sha, name
        }
    }'
}
```

---

## Resolving HEAD

```bash
resolve_head() {
    local head; head=$(cat .git/HEAD)
    case "$head" in
        ref:\ *)
            local ref="${head#ref: }"
            cat ".git/$ref" 2>/dev/null | tr -d '\n'
            ;;
        *)
            printf '%s' "$head" | tr -d '\n'
            ;;
    esac
}

current_branch() {
    local head; head=$(cat .git/HEAD)
    case "$head" in
        ref:\ refs/heads/*) printf '%s' "${head#ref: refs/heads/}" | tr -d '\n' ;;
        *) printf 'HEAD' ;;
    esac
}
```

---

## Reading .git/config for remote URL

```bash
remote_url() {
    local name="${1:-origin}"
    # Extract the url= line inside [remote "name"] section
    awk '/^\[remote "'"$name"'"\]/{found=1} found && /url *=/{gsub(/.*url *= */,""); print; exit}' \
        .git/config
}

# Parse github owner/repo from SSH or HTTPS URL
parse_github_owner_repo() {
    local url="$1"
    case "$url" in
        git@github.com:*)
            local path="${url#git@github.com:}"
            printf '%s\n' "${path%.git}"
            ;;
        https://github.com/*)
            local path="${url#https://github.com/}"
            printf '%s\n' "${path%.git}"
            ;;
    esac
    # Caller: owner=$(parse_github_owner_repo "$url" | cut -d/ -f1)
    #         repo=$(parse_github_owner_repo  "$url" | cut -d/ -f2)
}
```

---

## Quick reference: write a complete git object

```bash
# Example: store a blob for file README.md
sha=$(git_hash_object blob README.md)
git_compress README.md ".git/objects/${sha:0:2}/${sha:2}"
echo "blob $sha"
```

```bash
# Example: store a commit (content already in tmpfile)
sha=$(git_hash_object commit "$tmpfile")
git_compress "$tmpfile" ".git/objects/${sha:0:2}/${sha:2}"
printf '%s\n' "$sha" > ".git/refs/heads/$(current_branch)"
```
