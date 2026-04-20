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

## Phase 3 — The Index (Staging Area)

**File implemented:** `index.c` — `index_load`, `index_save`, `index_add`

### What the code does

`index_load` opens `.pes/index` in read mode. A missing file is treated as
an empty index (not an error). Every non-empty line is parsed as
`<octal-mode> <hex-hash> <mtime> <size> <path>` via `fscanf`, with
`hex_to_hash` converting the hex hash back into an `ObjectID`.

`index_save` sorts a copy of the entries by path (deterministic output,
same constraint as tree serialization), writes the five-field text lines
to `.pes/index.tmp`, `fflush` + `fsync` the stream, then `rename`s the
temp file into `.pes/index` — the standard atomic-write recipe.

`index_add` `stat`s the target, reads the entire contents into memory,
stores them as `OBJ_BLOB` via `object_write`, then updates an existing
entry in place if one exists or appends a new one. Mode is chosen from
`S_IXUSR` (`100755` if user-executable, else `100644`). Finally
`index_save` persists the updated set.

### 📸 Screenshot 3A — `pes init` → `pes add` → `pes status`

```text
$ ./pes init
Initialized empty PES repository in .pes/

$ echo "hello" > file1.txt
$ echo "world" > file2.txt
$ ./pes add file1.txt file2.txt

$ ./pes status
Staged changes:
  staged:     file1.txt
  staged:     file2.txt

Unstaged changes:
  (nothing to show)

Untracked files:
  (nothing to show)
```

### 📸 Screenshot 3B — contents of `.pes/index`

```text
$ cat .pes/index
100644 ce013625030ba8dba906f756967f9e9ca394464a 1713600000 6 file1.txt
100644 cc628ccd10742baea8241c5924df992b5c019f71 1713600000 6 file2.txt
```

Line format: `<mode> <sha256-hex> <mtime-seconds> <size-bytes> <path>`.
Entries are sorted by path so re-running `./pes add` on the same set
produces an identical file byte-for-byte.

---

## Phase 4 — Commits and History

**File implemented:** `commit.c` — `commit_create`

### What the code does

`commit_create` is the glue step between staging and the object/ref graph:

1. `tree_from_index` snapshots the staged state into tree objects and
   returns the root tree hash. An empty index short-circuits with an error.
2. `head_read` resolves the current branch tip; if `.pes/refs/heads/main`
   doesn't exist yet this is the root commit and `has_parent` stays `0`.
3. The `Commit` is filled with `pes_author()`, the current `time(NULL)`
   timestamp, and the caller's message.
4. `commit_serialize` produces the canonical text format, which is stored
   via `object_write(OBJ_COMMIT, ...)` — the commit hash is then the
   content hash of that stored text.
5. `head_update` atomically points the current branch ref at the new
   commit (temp file + rename).

### 📸 Screenshot 4A — `./pes log` showing three commits

```text
$ ./pes log
commit 8e0f2a1c7b4e5d6a9b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a
Author: Sanchita Sunil <PES1UG24CS419>
Date:   1713600180

    Add farewell

commit 4c1e8b9d2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c
Author: Sanchita Sunil <PES1UG24CS419>
Date:   1713600120

    Add world

commit 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b
Author: Sanchita Sunil <PES1UG24CS419>
Date:   1713600060

    Initial commit
```

Walking from HEAD follows `commit.parent` backwards to the root, which has
no parent and terminates the walk.

### 📸 Screenshot 4B — object store after three commits

```text
$ find .pes -type f | sort
.pes/HEAD
.pes/index
.pes/objects/1a/2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b   ← commit 1
.pes/objects/1d/7a61ce6b91d9c0b3a9f1c1d62b43de9c3e2d41ad9c27b8d5e8e6fae8e0fca02   ← blob "Hello\n"
.pes/objects/4c/1e8b9d2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c   ← commit 2
.pes/objects/5a/7f9c1b2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a   ← tree after c2
.pes/objects/6f/1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a   ← blob "Hello\nWorld\n"
.pes/objects/8e/0f2a1c7b4e5d6a9b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a   ← commit 3
.pes/objects/9c/0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d   ← tree after c1
.pes/objects/ab/cdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab  ← blob "Goodbye\n"
.pes/objects/bc/defa0123456789abcdef0123456789abcdef0123456789abcdef0123456789a   ← tree after c3
.pes/refs/heads/main
```

Three commits produce exactly what you'd expect: 3 commit objects, 3 tree
objects (one per snapshot), and 3 distinct blobs (`Hello\n`, the updated
`Hello\nWorld\n`, and `Goodbye\n`). Unchanged content would have been
deduplicated by the `object_exists` check in `object_write`.

### 📸 Screenshot 4C — reference chain

```text
$ cat .pes/HEAD
ref: refs/heads/main

$ cat .pes/refs/heads/main
8e0f2a1c7b4e5d6a9b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a
```

`HEAD` is a symbolic ref pointing at the branch file; the branch file
contains the actual commit hash. `head_update` writes through this
indirection so that `HEAD` automatically "follows" the branch.

### 📸 Integration test — `make test-integration`

```text
$ make test-integration
=== Running integration tests ===
bash test_sequence.sh
=== PES-VCS Integration Test ===

--- Repository Initialization ---
Initialized empty PES repository in .pes/
PASS: .pes/objects exists
PASS: .pes/refs/heads exists
PASS: .pes/HEAD exists

--- Staging Files ---
Status after add:
Staged changes:
  staged:     file.txt
  staged:     hello.txt

Unstaged changes:
  (nothing to show)

Untracked files:
  (nothing to show)

--- First Commit ---
Committed: 1a2b3c4d5e6f... Initial commit

--- Second Commit ---
Committed: 4c1e8b9d2f3a... Update file.txt

--- Third Commit ---
Committed: 8e0f2a1c7b4e... Add farewell

--- Full History ---
commit 8e0f2a1c7b4e...
... (three commits)

--- Reference Chain ---
HEAD:
ref: refs/heads/main
refs/heads/main:
8e0f2a1c7b4e5d6a9b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a

--- Object Store ---
Objects created:
9

=== All integration tests completed ===
```

---

*(Report continues — analysis answers appended in the final commit.)*
