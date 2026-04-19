# Build PES-VCS — A Version Control System from Scratch

**Name:** Sinchana Kulkarni  
**SRN:** PES2UG24CS504  

---

## Table of Contents

1. [Phase 1: Object Storage Foundation](#phase-1-object-storage-foundation)
2. [Phase 2: Tree Objects](#phase-2-tree-objects)
3. [Phase 3: The Index (Staging Area)](#phase-3-the-index-staging-area)
4. [Phase 4: Commits and History](#phase-4-commits-and-history)
5. [Final: Integration Test](#final-integration-test)
6. [Phase 5: Branching and Checkout — Analysis](#phase-5-branching-and-checkout--analysis)
7. [Phase 6: Garbage Collection — Analysis](#phase-6-garbage-collection--analysis)
8. [Implementation Summary](#implementation-summary)

---

## Phase 1: Object Storage Foundation

### Screenshot 1A — ##`./test_objects` output (all tests passing)

<img width="1078" height="195" alt="Screenshot 2026-04-18 at 18 34 57" src="https://github.com/user-attachments/assets/e55f57f3-3378-441f-9e04-8926716911af" />


### Screenshot 1B — ##`find .pes/objects -type f` (sharded directory structure)

<img width="930" height="115" alt="Screenshot 2026-04-18 at 20 17 04" src="https://github.com/user-attachments/assets/b53e8106-38d1-4d3f-996c-c627c3fb70df" />


---

## Phase 2: Tree Objects

### Screenshot 2A — ##`./test_tree` output (all tests passing)

<img width="697" height="180" alt="Screenshot 2026-04-18 at 19 10 02" src="https://github.com/user-attachments/assets/6c518eb7-3c28-4e82-866e-11c7af4c8f06" />


### Screenshot 2B — ##`xxd` of a raw tree object (first 20 lines)

<img width="1371" height="53" alt="Screenshot 2026-04-18 at 20 20 05" src="https://github.com/user-attachments/assets/39328e8b-af44-490b-8c1e-bf5f26f634dd" />


---

## Phase 3: The Index (Staging Area)

### Screenshot 3A — ##`pes init` → `pes add` → `pes status` sequence

<img width="931" height="716" alt="Screenshot 2026-04-19 at 16 41 01" src="https://github.com/user-attachments/assets/205c7fe7-4ab9-4927-8d54-f01136d03609" />


### Screenshot 3B — ##`cat .pes/index` (text-format index with entries)

<img width="1053" height="118" alt="Screenshot 2026-04-19 at 16 41 06" src="https://github.com/user-attachments/assets/925729f7-a5a9-4521-bb76-681aba1b390f" />


---

## Phase 4: Commits and History

### Screenshot 4A — ##`pes log` output with three commits

<img width="860" height="464" alt="Screenshot 2026-04-19 at 18 16 19" src="https://github.com/user-attachments/assets/08bb3c5f-c8b6-4a78-861c-f582c01416a8" />


### Screenshot 4B — ##`find .pes -type f | sort` (object store growth after 3 commits)

<img width="937" height="380" alt="Screenshot 2026-04-19 at 18 19 13" src="https://github.com/user-attachments/assets/22d4c2d8-28fa-44dd-b2ad-38ad4c5f47a8" />


### Screenshot 4C — ##`cat .pes/refs/heads/main` and `cat .pes/HEAD`

<img width="789" height="108" alt="Screenshot 2026-04-19 at 18 19 41" src="https://github.com/user-attachments/assets/a79e7375-7152-4fe9-bd31-c9f100096333" />


---

## Final: Integration Test

### Screenshot — ##Full integration test (`make test-integration`) — Part 1

<img width="1059" height="751" alt="Screenshot 2026-04-19 at 19 16 47" src="https://github.com/user-attachments/assets/ff4a1f80-9d80-42fb-aefe-b28681358f46" />


### Screenshot — ##Full integration test (`make test-integration`) — Part 2

<img width="1239" height="728" alt="Screenshot 2026-04-19 at 19 19 05" src="https://github.com/user-attachments/assets/8aa325af-93a7-4b52-8d86-e4cc9cbb5f73" />


---

## Phase 5: Branching and Checkout — Analysis

### Q5.1 — How would you implement `pes checkout <branch>`?

To implement checkout, three things must happen in sequence:

1. **Update `.pes/HEAD`** to contain `ref: refs/heads/<branch>` so the repo knows which branch is active.
2. **Reconstruct the working directory** — read the target branch's commit, walk its tree object recursively, and for each blob entry write the file content back to disk, overwriting files that differ.
3. **Update the index** to exactly reflect the checked-out branch's files (paths, hashes, modes, and metadata).

The operation is complex for several reasons. First, you must handle files that exist in the current branch but not the target — these must be deleted from the working directory. Second, you must recursively walk tree objects to handle nested subdirectories. Third, any uncommitted local changes that would be overwritten must be detected and rejected before making any changes, so the operation is safe to abort cleanly.

---

### Q5.2 — Detecting a dirty working directory conflict

The algorithm compares three sources of truth: the index, the working directory files on disk, and the target branch's tree.

For each file tracked in the current index:
1. `stat()` the file on disk and compare `mtime` and `size` to the index entry. If they differ, the file has been modified since staging — it is "dirty".
2. Look up that same file path in the target branch's tree (by walking the tree object). If the blob hash in the target branch differs from the blob hash in the current index, switching branches would overwrite the user's local change.
3. If both conditions are true — the file is dirty AND it differs between branches — refuse checkout and print an error.

This approach uses only the index and the object store, with no external tools.

---

### Q5.3 — Detached HEAD and recovering lost commits

In detached HEAD state, `.pes/HEAD` contains a raw commit hash directly instead of `ref: refs/heads/branchname`. New commits are still created normally and chained together, but since no branch file is being updated, once you switch away (e.g. `pes checkout main`), HEAD stops pointing at those commits.

The commits are not immediately deleted — they still exist in the object store as long as garbage collection has not run. To recover them, the user would need to know the hash of the last commit made in detached HEAD state. They could then create a new branch pointing to it:

```
pes branch recovery-branch <commit-hash>
```

In a real implementation, a reflog (a log of every position HEAD has pointed to) would make this trivial to recover automatically.

---

## Phase 6: Garbage Collection — Analysis

### Q6.1 — Algorithm to find and delete unreachable objects

**Algorithm — Mark and Sweep:**

**Mark phase:** Start from every branch ref file in `.pes/refs/heads/` and from HEAD. For each reachable commit hash, parse the commit object to get its tree hash and parent hash. Add all of these to a "reachable" set. Then recursively walk each tree object, adding every blob and subtree hash to the reachable set. Continue until no new hashes are discovered.

**Sweep phase:** Walk every file under `.pes/objects/` using `opendir`/`readdir`. Reconstruct the hash from each file's directory name and filename. If that hash is not in the reachable set, delete the file.

**Data structure:** A hash set (implemented as a hash table or a sorted array with binary search) of 32-byte `ObjectID` values gives O(1) or O(log n) lookup during the sweep.

**Estimation for 100,000 commits, 50 branches:** Assuming an average of 50 files and 10 tree objects per commit, you would visit approximately 100,000 commits + 1,000,000 trees + 5,000,000 blobs = roughly **6 million objects** in the mark phase.

---

### Q6.2 — Race condition between GC and a concurrent commit

**The race condition:**

1. GC scans all reachable objects and builds its reachable set. At this moment, a new blob X has not been written yet.
2. `pes add` runs concurrently and writes blob X to `.pes/objects/`.
3. `pes commit` starts building a tree that references blob X, but has not yet written the commit object.
4. GC's sweep phase runs and **deletes blob X** because it was not in the reachable set computed in step 1.
5. The commit finishes and writes a tree + commit object that permanently references the now-deleted blob X. The repository is silently corrupted.

**How Git avoids this:** Git's garbage collector uses a **grace period** — any object file newer than a fixed threshold (default: 2 weeks) is never deleted, even if it appears unreachable. Since `add` and `commit` operations complete in milliseconds to seconds, this ensures no in-progress operation's objects can be swept. Git also uses lock files during pack operations to signal that a write is in progress, causing GC to back off.

---

## Implementation Summary

| File | Status |
|---|---|
| `object.c` | Complete — `object_write` and `object_read` implemented |
| `tree.c` | Complete — `tree_from_index` implemented |
| `index.c` | Complete — `index_load`, `index_save`, `index_add`, `index_status` implemented |
| `commit.c` | Complete — `head_read`, `head_update`, `commit_create` implemented |
| `pes.c` | Complete — `cmd_commit` implemented |
