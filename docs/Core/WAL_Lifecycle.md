# WAL Lifecycle

### 0.1 Segment States

```java
CREATED → ACTIVE → SEALED → OBSOLETE → DELETED
```

---

### 0.2 State Definitions

**CREATED**
- File created

**ACTIVE**
- Receives writes
- Only one one active segment

**SEALED**
- No further writes
- Must be structurally complete

**OBSOLETE**
Condition:
```java
segment_max_seq <= flush_persisted_seq
```

**DELETED**
- Removed permanently

---

#### 0.2.1 Segment Lifecycle Transitions

- `CREATED → ACTIVE` immediately upon initialization
- `ACTIVE → SEALED` only via rotation policy
- `SEALED → OBSOLETE` when:
  - segment_max_seq ≤ flush_persisted_seq
- `OBSOLETE → DELETED` via background cleanup

**Transition Rules**
- Transitions must be monotonic
- Atomic at WAL append boundary
- No state may be skipped
- State transitions must preserve recovery correctness

---

### 0.3 Flush Watermark

```java
flush_persisted_seq = max seq persisted to SST
```

---

### 0.4 WAL ↔ Memtable

- WAL append must occur before memtable apply
- Memtable state reflects WAL order
- Memtable may span multiple segments
- Deletion is flush-based

---

#### 0.4.1 Visibility Rule

- A write becomes visible after memtable apply
- Visibility does not imply durability

---

## 1. Segment Rotation Policy

WAL segments are rotated to bound file size and maintain append efficiency.

---

### 1.1 Rotation Triggers

A segment transitions from `ACTIVE → SEALED` when:

1. Segment size reaches configured maximum, OR
2. Engine initiates manual rotation (e.g., shutdown)

---

### 1.2 Properties

- Rotation creates a new `ACTIVE` segment
- Segment file numbers are strictly increasing
- No gaps in segment numbering

---

## 2. WAL Preallocation

### 2.1 Semantics
- Preallocation = reserved capacity only
- Not valid WAL data

---

### 2.2 Boundary
- Only written records are valid
- Unwritten space is ignored

---

### 2.3 Recovery Rule
- Ignore preallocated/unwritten regions

---

### 2.4 Crash Safety
- Partial preallocation does not affect correctness


---

## 3. WAL Metadata Model

### 3.1 Metadata Fields

| Field           | Description                               |
|-----------------|-------------------------------------------|
| file_number     | segment ID                            |
| segment_min_seq | first seq                           |
| segment_max_seq | last seq                             |
| state           | lifecycle states     |
| size_bytes      | size     |

---

### 3.2 Persistent Model

- Metadata is not persisted separately
- Reconstructed from WAL

---

### 3.3 Source of Truth

> WAL is the only source of truth

---

### 3.4 Trade-off

- Simpler correctness
- Higher recovery cost

---

## 4. Manifest ↔ WAL Coordination

### 4.1 Replay Boundary

```java
Replay segments where:
segment_number ≥ manifest_log_number
```

---

### 4.2 Manifest Advancement Rule

Advance only when:
- Data is persisted to SST
- SST is durable

---

### 4.3 Ordering Constraint

```java
WAL → SST → Manifest
```

---

### 4.4 Safety Guarantee

- No data loss
- No duplication

---

### 4.5 Deletion Coupling

WAL can be deleted only if:
- **OBSOLETE**
- Below manifest boundary

---

## 5. Sequence Recovery

- Restore global sequence = max observed in WAL
- No external metadata required

---

### Invariants

- Exactly one WAL segment is `ACTIVE` at any time
- Segment states transition monotonically: `CREATED` → `ACTIVE` → `SEALED` → `OBSOLETE` → `DELETED`
- A segment is deleted only after `segment_max_seq ≤ flush_persisted_seq`
- Manifest boundary never excludes required WAL for recovery
- WAL → SST → Manifest ordering is strictly enforced
- Only the last segment may contain an incomplete tail; sealed segments must be complete
- Preallocated or unwritten space is never treated as valid WAL data
- All WAL metadata must be reconstructible solely from WAL contents