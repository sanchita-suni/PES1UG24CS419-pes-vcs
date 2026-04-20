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

*(Report continues — Phase 2, 3, 4, and analysis sections appended in subsequent commits.)*
