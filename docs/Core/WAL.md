# WAL (Write-Ahead Log) v1.0.0

### WAL Guarantees

- No acknowledged write is lost (based on sync policy)
- No partial write is appliied
- Deterministic recovery
- Total ordering of writes
- Corruption is detectable
- Idempotent replay
- WAL is the **source of truth**

---

### Layered Architecture

```java
Layer A → Physical Log (segments, blocks, fragments)
Layer B → Commit Pipeline (ordering, batching, serialization)
Layer C → Durability Policy (fsync + ACK semantics)
```

---

## 1. Layer A - Physical Log Format

### 1.1 Segment Model

```python
wal-000001.log
wal-000002.log
wal-000003.log
```

- Monotonic, zero-padded, lexicographically ordered
- Fixed-sized segments (configurable)
- Append-only

Each segment:

```go
Segment
 ├── Block (32 KB)
 ├── Block (32 KB)
 ├── Block (32 KB)
 └── ...
```

- Fixed Size blocks (`32 KB`)
- Records may span blocks via fragmentation

---

### 1.2 Record Model

- Each WAL record = **Serialized WriteBatch**
- WriteBatch = atomic group of operations

---

### 1.3 Fragmentation

- Large WriteBatch may span blocks using fragments:<br>
  - `FULL`<br>
  - `FIRST → MIDDLE → LAST`<br>

**Fragment Integrity Constraints**

- Fragment sequence must be valid
- Fragments must be contiguous
- Partial fragment chains are invalid

---

### 1.4 Valid Data Boundary

- File size ≠ valid WAL data
- Valid WAL = **only successfully written records**

Logical end of WAL = last valid record 

All bytes beyond this boundary are invalid.

---

### 1.5 Log Addressing Model (LSN)

The WAL defines a logical and physical addressing scheme for records.

---

#### 1.5.1 Logical Address (Sequence Number)

- Each operation is assigned a monotonically increasing **sequence number**
- Defines total ordering of writes

---

#### 1.5.2 Physical Address (Log Sequence Number - LSN)

Each WAL record is uniquely identified by **LSN**.

where:

```
LSN = (segment_file_number, byte_offset)
```

---

#### 1.5.3 Properties

- LSN is strictly increasing in append order
- LSN defines the physical position of a record in WAL
- Sequence number defines logical ordering
- Both must remain consistent

---

### 1.6 WriteBatch Size Constraint

- WriteBatch must fit within a single WAL segment

**Contract**
- `WriteBatch_size ≤ WAL_segment_size`

**Oversized Handling**
- Oversized batches are **rejected**
- No splitting, spilling, or implicit chunking

---

### 1.7 Record Validity Definition

A record is valid only if:

1. CRC check passes
2. Length is consistent
3. Fragment sequence is valid
4. Fragment chain is complete
5. Record is fully reconstructible

CRC alone is insufficient.

---

## 2. Layer B - Commit Pipeline

### 2.1 Architecture

```go
Client Threads
    ↓
Write Queue
    ↓
WAL Writer 
    ↓
Layer A 
    ↓
Memtable
```

Single execution context:
- Assigns sequence numbers
- Appends WAL
- Applies memtable
- Controls sync
- Releases writers

---

### 2.2 Write Lifecycle

1. Validate WriteBatch
2. Enqueue
3. Batch formation
4. Sequence assignment
5. WAL append
6. Memtable apply
7. Sync (Layer C)
8. ACK

---

### 2.3 Write Admission Validation

Before entering pipeline:
- Size ≤ segment size
- Structurally valid

Invalid writes are rejected before WAL interaction.

---

### 2.4 Backpressure & Admission Control

**Bounded Queue**
- Write queue must be bounded (count or memory)
- Unbounded growth is not allowed

**Backpressure Trigger**
- Queue full OR
- WAL throughput < incoming rate

**Admission Control**
- Writers must:
  - Block, or
  - Be rejected

**Propogation**
Backpressure propagates to clients:
- Increased latency
- Controlled throughput

---

### 2.5 Throughput Constraint


System throughput is bounded by **WAL write + sync cost**

---

### 2.6 Design Goals

1. Strict sequence monotonicity
2. WAL order == sequence order
3. Atomic batching
4. Deterministic crash boundary
5. **Bounded admission + backpressure**

---

## 3. Layer C — Durability Policy

### 3.1 Role
Defines:
- When a write becomes durable
- When ACK is issued

---

### 3.2 Sync Policies

(configurable)
1. **Per Bacth**
2. **Time-Based**
3. **Size-Based**
4. **Hybrid**

Policies affect timing, not definition of durability.

----

### 3.3.Durability Boundary Definition

A write is durable when:
- WAL record is persisted to stable storage
- Persistence survives crash
- Required filesystem metadata is persisted (if applicable)


---

### 3.4 Persistance Scope

#### 3.4.1 Persistence Ordering

Durability requires:

- WAL write must complete before durability operation
- Durability operation must guarantee persistence to stable storage

---

#### 3.4.2 Scope Clarification

Durability applies to:

- WAL file contents
- Required filesystem metadata (if segment created)

Durability does not depend on:

- In-memory state
- OS page cache visibility

---

### 3.5 Segment Creation Durability

If write occurs in new segment:
- Segment must be recoverable after crash

---

### 3.6 ACK Contract

ACK implies:<br>
> Write satisfies durability boundary under chosen policy

No premature ACK is allowed.

---

### 3.7 Visibility vs Durability

- Writes visible after memtable insert
- May be lost if not durable (sync policies)

---

### 3.8 Cross-Component Ordering
Strict ordering:
```java
WAL durability → SST durability → Manifest persistence
```

This ordering is mandatory.

---

## 4. Recovery Model

### 4.1 Replay Procedure

1. List segments in order
2. Filter using manifest boundary
3. Scan sequentially
4. Validate records
5. Stop at boundary
6. Replay valid records

---

### 4.2 Record Validation During Recovery

Recovery validates:
- CRC
- Length
- Fragment structure
- Fragment continuity
- Complete reconstruction

---

### 4.3 Partial Write Detection

Detect:
- Torn writes
- Truncated records
- Garbage tail
- Incomplete fragments

---

### 4.4 Padding vs Data

- Zero padding = end-of-data
- Not a valid record

---

### 4.5 Recovery Boundary Semantics

Two cases:

**(A) Corruption**
- Occurs in written region
- Treated as failure boundary

**(B) Incomplete Tail**
- Occurs at end of final segment
- Treated as valid termination

---

### 4.6 Segment-Specific Behavior

| Segment Type | Behavior |
|--------------|----------|
| **Sealed**   | Must be complete; invalid = corruption |
| **Last (Active)**   | May be incomplete tail |

---

### 4.7 Boundary Rule
- Stop at first invalid record
- Interpret based on segment type

---

### 4.8 Determinism
- Same WAL → same result
- No ambiguity allowed

---

### 4.9 Failure Semantics

The system must define behavior under IO and runtime failures.

---

#### 4.9.1 WAL Write Failure

If WAL append fails:

- The write must not be acknowledged
- The system may:
  - Retry, or
  - Propagate error to caller

---

#### 4.9.2 Sync Failure

If durability operation fails:

- No write in the affected commit group may be acknowledged
- The system must treat durability as not achieved

---

#### 4.9.3 Partial Failure

- Partial WAL writes are handled via recovery validation
- No partial record may be replayed

---

#### 4.9.4 System Safety

At no point:

- A failed write is acknowledged
- A non-durable write is treated as durable

---

#### 4.9.5 Crash Behavior

- Crash recovery rules define final state
- No additional guarantees are assumed beyond WAL durability

---

### Invariants

- WAL is strictly append-only and totally ordered (append order = sequence order = LSN order)
- Each WAL record is a complete, valid WriteBatch or ignored entirely (no partial replay) 
- Only structurally valid records (CRC + length + fragments) are eligible for replay
- Recovery always stops at the first invalid boundary (deterministic outcome)
- No acknowledged write violates the defined durability boundary (per sync policy)
- WAL is the single source of truth; all state must be derivable from it
- WriteBatch size never exceeds WAL segment size; no implicit splitting
- Write queue is bounded and admission control is enforced under load