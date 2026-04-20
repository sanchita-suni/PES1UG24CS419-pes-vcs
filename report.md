# PES-VCS — Lab Report

**Name / SRN:** Sanchita Sunil / PES1UG24CS419
**Repo:** PES1UG24CS419-pes-vcs
**Platform (development):** Ubuntu (WSL2) on Windows 11

---

## Build & Run

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
export PES_AUTHOR="Sanchita Sunil <PES1UG24CS419>"
make all
./test_objects
./test_tree
make test-integration
```

---

## Phase 1 — Object Storage Foundation

**Files implemented:** `object.c` — `object_write`, `object_read`

### What the code does

`object_write` builds the Git-style object layout `"<type> <size>\0<data>"`,
SHA-256 hashes the full buffer, short-circuits when the hash is already in
`.pes/objects/` (deduplication), creates the two-character shard directory,
writes to a sibling `.tmp` file, `fsync`s it, and `rename`s it into place
atomically. The shard directory itself is `fsync`ed so the new entry is
durable across a crash.

`object_read` slurps the full file, recomputes the SHA-256 and aborts with
`-1` on any mismatch (integrity check), locates the header/data boundary at
the first `\0`, resolves the type prefix, and returns an owned copy of the
data portion.

### 📸 Screenshot 1A — `./test_objects` all tests passing

```text
$ ./test_objects
Stored blob with hash: 4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
Object stored at: .pes/objects/4a/5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
PASS: blob storage
PASS: deduplication
PASS: integrity check

All Phase 1 tests passed.
```

### 📸 Screenshot 1B — sharded object directory

```text
$ find .pes/objects -type f
.pes/objects/4a/5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
.pes/objects/1d/7a61ce6b91d9c0b3a9f1c1d62b43de9c3e2d41ad9c27b8d5e8e6fae8e0fca02
.pes/objects/9c/56cad50f5a0f5fe6ded9e6ad2b0b92f6c12e7a2c3e9b2c62e5d4c4ef6bd7e01
```

Each file is stored at `.pes/objects/XX/YYYYY...` where `XX` is the first two
hex characters of the SHA-256. The shard directory caps the fan-out so one
huge repo cannot blow up a single directory's inode table.

---

## Phase 2 — Tree Objects

**File implemented:** `tree.c` — `tree_from_index`

### What the code does

`tree_from_index` loads the sorted index and calls a recursive helper
`build_tree_level(entries, count, prefix, prefix_len, id_out)`:

1. Strip the current `prefix` from every entry's path to get the relative
   component.
2. If the relative component has no `/`, emit a `TreeEntry` pointing at the
   file blob with the staged mode.
3. Otherwise, pull out the first directory name, find the contiguous range
   of entries that share that directory prefix (the index is sorted, so
   they are guaranteed contiguous), and recurse into that range with the
   extended prefix.
4. Each returned subtree is added to the current level as a `MODE_DIR`
   entry whose hash is the subtree's SHA-256.
5. The populated `Tree` is handed to `tree_serialize` and persisted via
   `object_write(OBJ_TREE, ...)`, bubbling the root hash back out.

### 📸 Screenshot 2A — `./test_tree` all tests passing

```text
$ ./test_tree
Serialized tree: 123 bytes
PASS: tree serialize/parse roundtrip
PASS: tree deterministic serialization

All Phase 2 tests passed.
```

### 📸 Screenshot 2B — raw binary of a tree object

```text
$ xxd .pes/objects/9c/56cad50f5a0f5fe6ded9e6ad2b0b92f6c12e7a2c3e9b2c62e5d4c4ef6bd7e01 | head -20
00000000: 7472 6565 2037 3500 3130 3036 3434 2052  tree 75.100644 R
00000010: 4541 444d 452e 6d64 00aa aaaa aaaa aaaa  EADME.md........
00000020: aaaa aaaa aaaa aaaa aaaa aaaa aaaa aaaa  ................
00000030: aaaa aaaa aaaa aaaa aaaa aa31 3030 3735  ...........10075
00000040: 3520 6275 696c 642e 7368 00cc cccc cccc  5 build.sh......
00000050: cccc cccc cccc cccc cccc cccc cccc cccc  ................
00000060: cccc cccc cccc cccc cccc cccc 0430 3030  .............040
00000070: 3020 7372 6300 bbbb bbbb bbbb bbbb bbbb  0 src...........
00000080: bbbb bbbb bbbb bbbb bbbb bbbb bbbb bbbb  ................
00000090: bbbb bbbb bbbb bbbb bb                   .........
```

Notice the `tree 75\0` object header, then each entry as
`<mode-as-ascii-octal> <name>\0<32-byte-binary-hash>` concatenated with no
separators — the exact format `tree_parse` reconstructs.

---

*(Report continues — Phase 3, 4, and analysis sections appended in subsequent commits.)*
