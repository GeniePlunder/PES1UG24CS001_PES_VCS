# PES-VCS: A Content-Addressable Version Control System

**Author:** Abhinav Agraharam
**SRN:** PES1UG24CS001

---

##  Project Overview
PES-VCS is a local version control system built from scratch in C, designed to demonstrate the intersection of **Operating Systems** and **Filesystems**. It implements a Git-like architecture using content-addressable storage, Merkle trees for directory snapshots, and a staged index for change detection.

---

## Technical Architecture

### Phase 1: The Object Store
The foundation of PES-VCS is a content-addressable store located in `.pes/objects`. 
- **Hashing:** Every file (blob), directory (tree), and commit is identified by a SHA-256 hash.
- **Atomicity:** To prevent data corruption, objects are written to a temporary file via `mkstemp()` and then moved to their final path using `rename()`.
- **Sharding:** Objects are stored in subdirectories named after the first two hex characters of their hash to prevent directory performance degradation.

### Phase 2: Tree Objects
Trees represent the project structure at a specific point in time. 
- **Recursion:** Implemented a recursive builder that groups index entries by directory prefix.
- **Serialization:** Trees are stored as binary objects containing file modes (octal), names, and 32-byte raw hashes.

### Phase 3: The Staging Area (Index)
The index acts as the "middleman" between the working directory and the repository history.
- **Metadata Tracking:** Stores `mtime` and `size` to perform fast diffs without re-hashing every file.
- **Atomic Persistence:** The index is rewritten atomically on every `pes add` or `pes remove` operation.

### Phase 4: Commits and History
Commits tie a tree snapshot to a specific author and point in time.
- **History Chain:** Each commit (except the first) contains a `parent` pointer, creating a directed acyclic graph (DAG) of project history.
- **Reference Updates:** The system updates `.pes/refs/heads/main` and shifts the `HEAD` pointer to the latest commit hash upon every successful commit.

---

##  Advanced Analysis

## Phase 5: Branching and Checkout (Analysis)

### Q5.1: Implementing `pes checkout <branch>`
To implement a branch checkout, the system must perform the following:
1. **Update Reference:** Modify `.pes/HEAD` to point to the chosen branch file in `refs/heads/`.
2. **Working Directory Sync:** The system must traverse the Tree object of the target commit. For every blob entry, it must write the contents back to the working directory.
3. **Destructive Cleanup:** Files currently in the working directory that are not present in the target branch's tree must be deleted.
**Complexity:** This is complex because it involves atomic replacement of the working directory state while ensuring that uncommitted local changes are not accidentally overwritten.

### Q5.2: Detecting "Dirty" Working Directories
A "dirty" directory contains modifications that aren't yet committed. We detect this by comparing:
- **Disk vs. Index:** If the `st_mtime` or `st_size` of a file on disk differs from the Index, it is **modified**.
- **Index vs. HEAD:** If the hash of a file in the Index differs from the hash in the current HEAD commit, it is **staged**.
If either of these conditions is true for a file that differs between the current branch and the target branch, the system should refuse the checkout.

### Q5.3: Detached HEAD State
A "Detached HEAD" occurs when `.pes/HEAD` contains a raw commit hash instead of a symbolic reference to a branch.
- **Committing:** Commits made in this state work normally, but only HEAD moves forward.
- **Recovery:** Since no branch points to these new commits, they can be "lost" if the user checks out another branch. To recover them, the user must manually create a new branch reference using the last known commit hash from the terminal output or log.

---

## Phase 6: Garbage Collection (Analysis)

### Q6.1: Reachability Algorithm
To reclaim space, we use a **Mark-and-Sweep** approach:
1. **Mark Phase:** Start from all pointers in `refs/heads/`. Recursively traverse every commit, its root tree, sub-trees, and blobs. Store all unique hashes encountered in a hash set.
2. **Sweep Phase:** Iterate through all files in `.pes/objects/`. If an object's hash is not in the "reachable" set, it is an orphan and can be deleted.
**Estimation:** In a repository with 100,000 commits, we only visit unique hashes. Thanks to Merkle tree deduplication, the number of objects visited is significantly lower than the total number of files across all snapshots.

### Q6.2: GC Race Conditions
Running GC during a commit is dangerous because of the "unreferenced object" window. A `commit` process may write a new blob to the store, but before it can update the branch reference to include that blob, the GC might see it as unreachable and delete it.
**Git's Solution:** Git uses a **grace period** (graceful pruning). Objects are only deleted if they are unreachable AND haven't been modified/created in the last 14 days.

---

## Usage Instructions

### Building
```bash
make all           # Build the pes binary and test suites
make clean         # Remove build artifacts
