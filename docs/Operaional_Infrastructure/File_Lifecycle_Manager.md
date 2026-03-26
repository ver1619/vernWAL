# File Lifecycle Manager

### File Lifecycle Guarantees

The File Lifecycle Manager must ensure:

- **No data loss** due to premature deletion
- **No file leaks** or orphan accumulation
- Deterministic file visibility via **metadata**
- **Safe coordination** with WAL, compaction, and iterators
- Crash-safe recovery of file state
- Strict separation of logical visibility and physical existence

---

## 1. Core Responsibility

The **File Lifecycle Manager** governs:

- Creation
- Activation
- Tracking
- Obsolescence
- Deletion

of all persistent storage files.

It defines the **physical safety boundary** of the storage engine.

---

## 2. File Types

### 2.1 WAL Files

Properties:
- Append-only
- Sequential writes
- Source of durability

Lifecycle constraint:
- Deleted only after **flush + durability boundary advancement**

---

### 2.2 SST Files

Properties:
- Immutable
- Created by flush or compaction

Lifecycle constraint:
- Visible only via manifest
- Deleted only when unreferenced

---

### 2.3 Manifest Files

Properties:
- Metadata log
- Append-only
- Defines database state

---

### 2.4 Temporary Files

Properties:
- Created during intermediate operations
- Not part of visible state

Lifecycle constraint:
- Must not be externally visible until committed

---

## 3. File Lifecycle State Model

Each file follows a strict state transition model:
```java
CREATED → ACTIVE → OBSOLETE → DELETABLE → DELETED
```

---

### 3.1 State Definitions

- **CREATED**
    - File exists on disk
    - Not yet visible
- **ACTIVE**
    - Referenced by current manifest
    - Visible to system
- **OBSOLETE**
    - No longer referenced by active manifest
    - May still be in use
- **DELETABLE**
    - Proven safe to delete
    - No references remain
- **DELETED**
    - Physically removed

---

### 3.2 Transition Rules

- **`CREATED → ACTIVE`**:
    - Only via manifest update
- **`ACTIVE → OBSOLETE`**:
    - When removed from manifest
- **`OBSOLETE → DELETABLE`**:
    - After all references are released
- **`DELETABLE → DELETED`**:
    - Background deletion

---

## 4. Visibility Model

### 4.1 Visibility Boundary
```java
A file is visible if it is referenced by the current manifest state
```

---

### 4.2 Source of Truth

```java
Manifest is the authoritative source of file visibility
```
- No file is considered part of database state without manifest reference

---

## 5. File Creation

### 5.1 Creation Rules
- File numbers are globally unique
- File numbers are never reused

---

### 5.2 Creation Constraint
Newly created files must remain invisible until:
- Fully written
- Successfully committed to manifest

---

## 6. Activation

A file becomes active only when:

- It is referenced by the manifest
- The manifest update is durable

---

## 7. Obsolete File Detection

A file becomes obsolete when:
```java
No active Version references the file
```
---

## 8. Deletion Safety Model

### 8.1 Deletion Eligibility

A file is deletable only if:
```java
1. Not referenced by any active Version
2. Not pinned by any iterator
3. Not required for recovery
```

---

### 8.2 Safety Guarantee
No file required for:
- Reads
- Iteration
- Recovery<br>
  may be deleted

---


## 9. WAL Deletion Rules

A WAL file may be deleted only if:
```java
1. All contained writes are persisted to SSTs
2. Flush boundary has advanced
3. Durability boundary has advanced
```

---

### 9.1 Constraint
WAL deletion must not violate:
- Durability guarantees
- Recovery correctness

---

## 10. SST Deletion Rules

An SST file may be deleted only if:
```java
1. It is not referenced by any active Version
2. It is not required by any iterator
```

---

### 10.1 Replacement Rule
SST files become obsolete only after:
- Being replaced via compaction
- Manifest update reflects replacement

---


## 11. Temporary File Handling

### 11.1 Visibility Constraint
```java
Temporary files must remain invisible until atomically promoted
```

---

### 11.2 Promotion Rule
A temporary file becomes active only after:
- Successful completion
- Manifest inclusion

---

## 12. Orphan File Cleanup

### 12.1 Definition

An orphan file is:

```java
A file present on disk but not referenced by manifest
```

---

### 12.2 Cleanup Procedure

- Scan storage directory
- Remove unreferenced files

---

### 12.3 Safety Constraints

Cleanup must not delete:
```java
- Files pending manifest commit
- Files required for WAL recovery
```

---

## 13. Interaction Contracts

### 13.1 With WAL
WAL deletion must respect:
```java
- Durability boundary
- Flush completion
```

---

### 13.2 With Manifest
Manifest defines:
```java
- File visibility
- File activation
- File obsolescence
```

---

### 13.3 With Iterators
- Iterators pin Versions
- Files referenced by pinned Versions must not be deleted

---

### 13.4 With Compaction
Old files become obsolete only after:
```java
- Manifest update
- Deletion must be deferred until safe
```

---

### 13.5 With Recovery
- Only manifest-referenced files are considered valid
- Other files are treated as orphan

---

## 14. Concurrency Model
- File creation, activation, and deletion may occur concurrently
- Deletion must be coordinated through background scheduling

**Guarantee**<br>
No race may lead to:
- Deletion of active file
- Visibility of incomplete file

---

## 15. Crash Semantics

### 15.1 Persistence Model
- File visibility is determined by manifest state
- Files may exist on disk without being visible

---

### 15.2 Recovery Rules

After crash:
```java
1. Load manifest
2. Treat referenced files as active
3. Treat others as orphan
```
---

### 15.3 Crash Safety Guarantee
- No visible file is lost
- No incomplete file becomes visible

---

## 16. Constraints
- File lifecycle must be **metadata-driven**
- File deletion must be **deferred and validated**
- File visibility must be **atomic with manifest updates**
- No file may transition states out of order

---

### Invariants

- Manifest is the **single source of truth** for file visibility
- A file is visible iff referenced by manifest
- File numbers are globally unique and never reused
- No file is deleted while:
    - referenced by a Version
    - pinned by an iterator
    - required for recovery
- WAL files are deleted only after durability boundary is satisfied
- Temporary files are never visible before promotion
- File lifecycle follows strict state transitions
- Orphan files are eventually removed
- No race condition may violate file safety
- Crash recovery reconstructs correct file set solely from manifest
