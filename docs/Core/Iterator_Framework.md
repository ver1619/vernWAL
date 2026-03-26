# Iterator Framework

### Iterator Guarantees

The Iterator Framework must ensure:

- **Globally sorted traversal** across all storage layers
- **Snapshot-consistent reads** (MVCC correctness)
- **Deterministic merge semantics** across sources
- **No version resurrection** during iteration
- **Stable view of data** despite concurrent writes and compaction
- **Resource safety** via version pinning
- **No mutation of underlying data**



---

## 1. Core Abstraction

### 1.1 Unified Iterator Interface

All storage layers expose a common iterator abstraction:
```java
Seek(key)     → position at first key ≥ target  
Next()        → advance to next entry  
Valid()       → check iterator validity  
Key()         → return current key  
Value()       → return current value  
```

---

### 1.2 Abstraction Goal

The framework provides:

- A **single logical ordered stream** over:
    - Memtables
    - SST files
    - Multiple LSM levels
- Independent of physical storage layout

---

## 2. Iterator Composition Model

### 2.1 Hierarchical Construction

Iterators are composed in layers:
```java
Memtable Iterator
SST Block Iterator
SST File Iterator
Level Iterator
Merging Iterator
```

Higher-level iterators combine lower-level iterators.

---

### 2.2 Global Ordering Rule

All iterators must obey:
```java
InternalKey comparator: (user_key ASC, seq DESC, type DESC)
```

This guarantees:

- Newest versions appear before older versions
- Deterministic ordering across sources


---

## 3. Iterator Lifecycle

### 3.1 Creation Contract

At read start:

- Iterator must:
    - Capture a **snapshot_seq**
    - Bind to a **Version**

This defines a **stable read view**

---

### 3.2 Lifetime Guarantees
- Iterator pins the Version it was created from
- Underlying SST files must not be deleted while iterator is active

---

### 3.3 Destruction

On iterator destruction:

- All owned resources must be released
- Version reference must be released

---

## 4. Resource Ownership

### 4.1 Ownership Rules

- Parent iterators **own child iterators**
- Merging iterator owns:
    - Child iterators
    - Merge ordering structure
- SST iterators own:
    - Active block iterators
- Iterator owns:
    - Version reference

---

### 4.2 Resource Safety Guarantee

- No resource outlives its owning iterator
- No dangling reference to storage objects is allowed

---

## 5. Iterator Types

### 5.1 Memtable Iterator
- Traverses in-memory entries
- Ordered by InternalKey comparator

Provides:
- Most recent data

---

### 5.2 SST Block Iterator
- Traverses entries within a single block

Operates only on:
- Loaded block data

```java
Block
 ├─ entry
 ├─ entry
 ├─ entry
```


---

### 5.3 SST File Iterator

Responsible for:

- Index lookup
- Block selection
- Block iterator creation


---

### 5.4 Level Iterator

#### L1+ (non-overlapping levels):

- At most **one SST file** per level participates
- No intra-level merge required

```java
Create SST iterator
     ↓
Iterate within file
```

---

#### L0 (overlapping files)

- Multiple SST files may overlap
- Files ordered:
```java
newest file → oldest file
```
- Requires merging across files

---

## 6. Merging Iterator

### 6.1 Role

Combines multiple child iterators into a **single ordered stream**

Source include:
```java
Active Memtable
Immutable Memtables
L0 SSTs
L1+ SSTs
```

---

### 6.2 Ordering Contract
- Ordering strictly follows InternalKey comparator
- Global ordering must be preserved across all sources


---

### 6.3 Seek Semantics

On `Seek(target)`:

- Each child iterator performs `Seek(target)`
- Merging iterator selects:
```java
smallest key across children
```

---

### 6.4 Invalid State Handling

An iterator is invalid if:
- It reaches end of keyspace
- Underlying source is exhausted

Rules:
- Invalid child iterators must be removed from merge set
- Client must check `Valid()` before accessing data

Example:
```java
Before:

Heap
 ├ Iterator A
 ├ Iterator B
 └ Iterator C

Iterator B exhausted

After:

Heap
 ├ Iterator A
 └ Iterator C
```

---

## 7. Visibility Model (MVCC)

### 7.1 Visibility Rule

For snapshot `S`:
```java
record.sequence_number ≤ S
```

---

### 7.2 Filtering Process

During iteration:

1. Skip all records with:
```java
seq > snapshot_seq
```
2. Identify first visible version per user key

---

### 7.3 First-Visible Rule
- The first visible record defines the correct result
- No further versions of that key may be returned

---

## 8. User-Key Deduplication

### 8.1 Deduplication Rule

Once a visible version of a `user_key` is emitted:
```java
All remaining internal keys with same user_key must be skipped
```

---

### 8.2 Guarantee
- Output contains **unique user keys only**
- Ensures consistent range scan semantics

---

## 9. Tombstone Semantics

### 9.1 Visibility

If first visible record is:
```java
DELETE
```

Then:
- Key is logically absent

---

### 9.2 Masking Rule

A visible tombstone must:
```java
Mask all older versions of the same key
```

---

## 10. Range Scan Semantics

### 10.1 Output Definition

Range scans produce:
- Ordered sequence of **unique user keys**
- Each key mapped to:
    - Latest visible value

---

### 10.2 Example

```java
Internal entries:

A@100
A@90
B@105
B@70

Output:
A → value@100
B → value@105
```

---

## 11. Compaction Iterator Mode

| Iterator Type	| Snapshot Filtering
|----------------|------------------|
| Read Iterator | Applied |
| Compaction Iterator | Not Applied |

---

### 11.1 Compaction Requirement
- Must process **all internal keys**
- Must not apply snapshot visibility filtering

---

## 12. Interaction Contracts

### 12.1 With Internal Key

- Must rely on:
```java
(user_key ASC, seq DESC, type DESC)
```

- Correctness depends on comparator consistency

---

### 12.2 With Snapshot Manager

- Must apply:
```java
record.seq ≤ snapshot_seq
```
- Must enforce:
  - first-visible rule
  - tombstone masking

---

### 12.3 With Storage Layers

- Must not mutate underlying data
- Must preserve ordering across transformations

---

## 13. Concurrency Model

- Iterators operate on a **stable Version**
- Concurrent operations allowed:
  - Writes
  - Flush
  - Compaction

No interference with active iterators.

---

## Iterator Hierarchy


```java
                 Client Iterator
                        │
                        ▼
                 Merging Iterator
                        │
     ┌──────────────────┼──────────────────┐
     ▼                  ▼                  ▼
 Memtable          Immutable       SST Levels Iterator
 Iterator          Memtables 
                    Iterator              │
                            ┌─────────────┼─────────────┐
                            ▼             ▼             ▼
                            L0            L1            L2+ 
```

Data sources are logically ordered from newest to oldest:

```java
→ Active Memtable
→ Immutable Memtables
→ Level 0 SSTs
→ Lower SST levels
```

**InternalKey ordering** ensures that newer versions dominate older ones
even when keys originate from different storage layers.

---

## 14. Crash Safety

Iterators rely on:

- Immutable SST files
- Immutable memtables
- Stable Version references

### Guarantee
- Iterator view remains valid during concurrent system activity
- No dependency on WAL or recovery process

---

## 15. Cost Model

Iterator performance depends on:
- L0 file count
- Number of levels
- Block cache hit rate
- Compaction effectiveness

---

## 16. Constraints
- Iterator must be **forward-progressing**
- Iterator must not:
  - revisit keys
  - emit duplicate user keys
- Iterator must maintain:
  - strict ordering
  - snapshot correctness

---

### Invariants

- Global iteration order is **strictly sorted by InternalKey**
- First visible version defines correct result
- No older version of a returned key is emitted
- Tombstones mask all older versions
- Snapshot visibility rule is strictly enforced
- Iterators do not modify underlying data
- Iterator view is stable under concurrent operations
- Version pinning prevents premature file deletion
- All child iterators obey same comparator
- No version resurrection is possible