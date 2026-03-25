# Snapshot Manager

**The snapshot system must ensure:**
- **Stable, consistent reads** under concurrent writes
- **Deterministic visibility rules** based on sequence numbers
- **No version resurrection** under any condition
- **Safe compaction pruning** without violating visibility
- **Zero blocking for reads** (no global read locks)
- **Minimal runtime overhead** (O(1) read-path cost)

---

## 1. Core Concept

### 1.1 Sequence-Based MVCC
- Every committed write is assigned a **globally monotonically increasing sequence number**
- Sequence numbers define:
  - **Total ordering of writes**
  - **Visibility across snapshots**

---

### 1.2 Snapshot Definition

A snapshot is defined by:

```java
snapshot_seq
```

Which represents:

```java
All records with sequence_number ≤ snapshot_seq are visible
```

- Snapshots are **immutable**
- Snapshot state is **fully represented by snapshot_seq**

---

## 2. Boundary Definitions

### 2.1 Visibility Boundary

A record is visible under snapshot `S` if:

```java
record.sequence_number ≤ S
```

---

### 2.2 Pruning Boundary (Compaction-Time)

The system maintains:

```java
oldest_active_snapshot_seq
```

A version is eligible for pruning if:

```java
record.sequence_number < oldest_active_snapshot_seq
```

---

### 2.3 Boundary Separation

- **Visibility boundary** governs read correctness
- **Pruning boundary** governs version garbage collection

These boundaries are independent and must not be conflated.

---

## 3. Snapshot Lifecycle

### 3.1 Creation
When a snapshot is requested:
- Capture current **global sequence number**
- Register snapshot in active snapshot set

---

### 3.2 Properties
- Snapshot is **immutable** after creation
- Snapshot reflects **only committed state**
- Snapshot does not affect write ordering

---

### 3.3 Release

When a snapshot is released:

- It is removed from active snapshot set
- `oldest_active_snapshot_seq` is updated

If no snapshots remain:

```java
oldest_active_snapshot_seq = current_global_sequence
```

---

### 3.4 Lifecycle Flow

```java
Create Snapshot
    ↓
Reads use snapshot_seq
    ↓
Compaction respects pruning boundary
    ↓
Snapshot released
    ↓
Pruning boundary advances
```

---

## 4. Visibility Semantics

### 4.1 Record Selection

For a given key:

- Records are evaluated in **Internal Key order**
- The first record satisfying visibility boundary is selected

---

### 4.2 Delete Handling
- If first visible record is `DELETE`:
  - Key is considered **non-existent**

---

### 4.3 Masking Rule

A visible `DELETE` must:

```java
Mask all older versions of the same key
```

No older version may be returned. 

---

## 5. Snapshot Tracking

### 5.1 Active Snapshot Set

The system maintains a set of active snapshots with:

- Ordered by `snapshot_seq`
- Head = **oldest snapshot**
- Tail = **newest snapshot**

---

### 5.2 Purpose

- Determines **pruning boundary**
- Enables **safe compaction decisions**

---

## 6. Concurrency Model

### 6.1 Snapshot Operations

- **Snapshot creation**:
  - Reads global sequence atomically
- **Snapshot registration/removal**:
  - Serialized under snapshot metadata lock

---

### 6.2 Read Path

- Each read operates on its own `snapshot_seq`
- Reads:
  - Do not block writes
  - Do not block other reads
  - Do not require global locks

---

### 6.3 Compaction

- Reads `oldest_active_snapshot_seq` under metadata lock
- Must respect pruning boundary

---

### 6.4 System Concurrency

The system guarantees:

Concurrent execution of:
- Reads
- Writes
- Snapshot operations
- Compaction

Without violating visibility or ordering.

---

## 7. Interaction Contracts

### 7.1 With WAL

- Snapshot sequence must reflect **committed (WAL-appended) state only**
- Snapshot must not observe:
  - Uncommitted writes
  - Non-durable writes beyond defined boundary

---

### 7.2 With Internal Key

- Visibility evaluation must follow:

```java
record.sequence_number ≤ snapshot_seq
```

- Ordering guarantees ensure:
  - First visible record is correct

---

### 7.3 With Compaction

Compaction must:

- Respect pruning boundary
- Preserve:
  - At least one visible version per key for all snapshots
- Retain tombstones when required for correctness

---

## 8. Crash Semantics

### 8.1 Persistence Model
- Snapshots are **in-memory only**
- Not persisted across restarts

---

### 8.2 Recovery Behavior

On restart:

- No active snapshots exist
- System initializes:
```java
oldest_active_snapshot_seq = current_global_sequence
```

---

### 8.3 Implication

- Compaction may proceed without historical constraints
- No snapshot recovery is required

---

### Constraints

- Snapshot must be **constant-time to evaluate**
- Snapshot must not introduce **global coordination on read path**
- Snapshot must not depend on:
  - External metadata
  - Persistent state
- Snapshot system must scale with:
  - High read concurrency
  - High write throughput

---

### Invariants

- Sequence numbers are **globally strictly monotonically increasing**
- Sequence assignment order matches **WAL order**
- Snapshot sequence is:
  - Immutable
  - ≤ global sequence at creation

- Visibility rule:
```java
record.sequence_number ≤ snapshot_seq
```

- Pruning rule:
```java
record.sequence_number < oldest_active_snapshot_seq
```

- Snapshots are ordered by **snapshot_seq**
- DELETE masks all older versions under a snapshot
- Compaction never removes:
  - A version visible to any snapshot
  - A required tombstone
- Snapshot operations do not affect:
  - Write ordering
  - WAL semantics
- No read requires global locking
- No version resurrection is possible