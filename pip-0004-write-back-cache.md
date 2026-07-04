---
pip: 4
title: Application-level write-back cache for state
author: zarah (@zarah-s)
type: Standards Track
status: Accepted
created: 2026-05-21
accepted: 2026-05-24
requires: 2, 3
---

> **Post-pivot canonical locations.** This PIP was authored against the pre-pivot engine workspace (archived at [pyde-net/archive](https://github.com/pyde-net/archive)). The post-pivot engine has since been re-cut at [pyde-net/engine](https://github.com/pyde-net/engine) and the canonical implementation lives in:
>
> - **`crates/state/src/cache.rs`** — the DashMap-backed value cache + `CacheEntry` state machine + the parallel JMT pending-batch buffer.
> - **`crates/state/src/flush.rs`** — the background flush task, three-signal policy, and auto-tune.
> - Wired into the state crate's existing `StateStore::read_slot` (PR-4) and `StateCommitter::commit` (PR-3) paths.
>
> Both PIP-2 ([#1](https://github.com/pyde-net/pips/pull/1)) and PIP-3 ([#2](https://github.com/pyde-net/pips/pull/2)) — the `requires` dependencies — are Accepted; their reference implementations have shipped.

## Abstract

Add an application-level **write-back cache** to the state layer. State writes from each wave land in an in-memory cache instead of being flushed to RocksDB synchronously; the cache accumulates writes across a configurable **warm window** of recent waves; a **lazy flush with auto-tune** drains the cache to RocksDB asynchronously when memory pressure, wall-clock thresholds, or wave-count thresholds are reached, or eagerly on graceful shutdown.

Reads consult the cache first (fastest path); on miss, they fall through to PIP-3's prefetch-warmed RocksDB block cache (warm path); on miss there, they hit RocksDB directly (cold path).

State commit (state-root computation + HardFinalityCert signing) **still happens every wave**, in-memory, against the cached state. The cache decouples *logical* state commit from *physical* disk flush. Crash recovery is via replay of unflushed waves against the last on-disk snapshot — the chain log is authoritative.

This is **v1 scope**, not future direction. It composes with PIP-2 (clustered keys) and PIP-3 (scheduler-level prefetch) to give Pyde a three-layer state performance stack: clustered layout → batched warm reads → in-memory write absorption.

## Motivation

PIP-2 and PIP-3 reduce the cost of *reading* state. They do nothing for *writing* state.

A storage-heavy wave that touches 50K slots — entirely realistic for a busy chain — issues 50K `sstore` operations. Each one currently goes through:
1. JMT update (in-memory tree manipulation + dual-hash recomputation up the affected path).
2. RocksDB `put` for the changed leaf and every changed internal node on the path.
3. Optional WAL fsync for durability.

Of those three, step 2 is the bottleneck. RocksDB writes amplify through LSM-tree compaction; the disk-side work can dominate wave processing latency under sustained write load. And we are paying that cost *every wave*, even when most of those writes are to slots that will be overwritten again in the next wave (e.g., per-block counters, oracle aggregations, AMM pool state during a swap burst).

A write-back cache absorbs that churn. Slots written N times across the warm window get flushed to disk once with their final value. Disk-side write amplification drops proportionally to write locality.

## Specification

### Cache shape

The cache has **two cooperating parts**: a value cache for slot writes and a JMT pending-batch buffer for tree-node writes. Both drain to RocksDB on the same flush trigger via a single atomic `WriteBatch`.

#### Value cache

`DashMap<SlotHash, CacheEntry>` keyed by slot hash, with per-entry metadata:

```rust
pub struct CacheEntry {
    pub value: SmallVec<[u8; 32]>,  // most slot values fit in 32 bytes
    pub state: EntryState,
    pub written_at_wave: WaveId,
    pub last_read_at_wave: WaveId,
}

pub enum EntryState {
    Clean,   // matches RocksDB; safe to evict
    Dirty,   // changed in-memory since last flush; must flush before eviction
    Pending, // currently being flushed; reads still served, writes block briefly
}
```

`DashMap` is chosen for its lock-free reads and per-shard write locking, which fits the access-list-parallel execution pattern (PIP-2 puts contract X's slots in a contiguous shard region, reducing cross-shard contention).

#### JMT pending-batch buffer

A parallel queue holds the `TreeUpdateBatch` produced by each wave's JMT update. The buffer keeps batches in wave order so the flush task can drain them deterministically:

```rust
pub struct PendingJmtBatch {
    pub wave_id: WaveId,
    pub new_root: RootHash,
    pub batch: TreeUpdateBatch,  // node + value-history records from jmt
}

pub struct JmtPendingQueue {
    queue: parking_lot::Mutex<VecDeque<PendingJmtBatch>>,
}
```

The queue is small (≤ warm-window waves of batches; typically 32) and append/drain operations are amortised O(1). A single `Mutex` is sufficient — appends and drains never interleave at high frequency.

**Why a parallel queue instead of widening the value-cache key type:** the value cache is hot-path read-on-every-sload; widening its key type to `enum { SlotHash, JmtNodeKey }` would force every value-cache lookup through an extra match arm and inflate the DashMap shard footprint with low-locality JMT-node entries that the read path doesn't query. Keeping the two structures separate preserves the cache's hash-map shape while still letting both kinds of writes flush atomically.

### Warm window

The cache retains all writes from the last **N** waves, where N is configurable (default: 32 waves). This window is the eviction guard: entries within the window stay resident regardless of memory pressure (they may yet be re-read by upcoming waves, especially in workloads with hot keys).

Entries older than the warm window are flushed to RocksDB and may be evicted from the cache. Eviction respects:
- **Clean** entries: evictable immediately.
- **Dirty** entries: flush first, then evict.
- **Pending** entries: wait for in-flight flush, then evict.

### Read path

```
sload(slot_hash) {
    // Layer 1: cache
    if let Some(entry) = cache.get(&slot_hash) {
        update entry.last_read_at_wave = current_wave;
        return entry.value;
    }

    // Layer 2: PIP-3-warmed RocksDB block cache (transparent to caller)
    // Layer 3: cold RocksDB read
    let value = rocksdb.get(&slot_hash);

    // Pull into the application cache so subsequent reads in the warm window are fast
    cache.insert(slot_hash, CacheEntry {
        value: value.clone(),
        state: EntryState::Clean,
        written_at_wave: 0,  // sentinel: came from disk, not from a write
        last_read_at_wave: current_wave,
    });

    return value;
}
```

### Write path

```
sstore(slot_hash, value) {
    cache.entry(slot_hash)
        .and_modify(|e| {
            e.value = value.clone();
            e.state = EntryState::Dirty;
            e.written_at_wave = current_wave;
        })
        .or_insert(CacheEntry {
            value,
            state: EntryState::Dirty,
            written_at_wave: current_wave,
            last_read_at_wave: current_wave,
        });
    // Note: no RocksDB write here. The flush is asynchronous and batched.
}
```

### State-root computation

State root computation happens every wave, as today. It operates against the cache's current view of state. Changes accumulated in the cache contribute to the dual-hash JMT update path; the new state root reflects all writes up to the current wave.

The cache is queried for slot values during JMT path rehashing; cache misses fall through to RocksDB as in normal reads.

The resulting `TreeUpdateBatch` is **enqueued** on `JmtPendingQueue` rather than written to `jmt_cf` immediately. It carries everything the flush task needs to persist the JMT side atomically (`node_batch.nodes()` for internal nodes + `node_batch.values()` for value-history records). The JMT itself remains authoritative in-memory between flushes: subsequent reads of older versions traverse the cached value-history records before falling through to `jmt_cf`.

### Lazy flush with auto-tune

A background flush task watches three signals and triggers a flush when any one fires:

```rust
pub struct FlushPolicy {
    pub max_dirty_entries: usize,         // e.g., 100,000 — memory pressure
    pub max_wall_clock_ms: u64,           // e.g., 5000 — staleness bound
    pub max_waves_since_flush: u64,       // e.g., 16 — wave-count bound
    pub disable_auto_tune: bool,          // false by default
}
```

Auto-tune (enabled by default) adjusts these thresholds based on observed conditions:
- If RocksDB is under high write contention, flush more aggressively (lower thresholds).
- If memory pressure is high, flush more aggressively.
- If sustained write rate is low, flush less aggressively (let more coalescing happen).
- If a graceful shutdown is requested, flush immediately and synchronously.

When a flush fires, the flush task:
1. Snapshots the set of `Dirty` value-cache entries **and** drains all `PendingJmtBatch` entries up to the snapshot's wave watermark (atomic with respect to in-flight writes).
2. Marks the value-cache entries `Pending`.
3. Writes the combined set to RocksDB in a single `WriteBatch` that touches `state_cf` (slot puts/deletes from the value cache) and `jmt_cf` (node + value-history records from every drained JMT batch). The cross-CF `WriteBatch` is the same atomicity primitive PR-3's `StateCommitter` already uses — extended to span N waves' worth of writes instead of 1.
4. Transitions the value-cache entries to `Clean` on success; drops the drained JMT batches from the queue.
5. Logs the flush event with metrics (slot entries flushed, JMT batches drained, total bytes flushed, wall-clock time, RocksDB ack latency).

Reads during flush are served from the value cache (the `Pending` value is the source of truth until flush completes) and from the JMT pending queue for historical-version queries. Writes during flush block briefly on the entry's lock if they target a `Pending` slot.

### Crash recovery

The cache and the JMT pending queue are both volatile; on crash, unflushed writes to `state_cf` and unflushed `TreeUpdateBatch` entries for `jmt_cf` are lost. **The chain log is authoritative.** Recovery on restart:

1. Read the last on-disk JMT version (the most recent successful flush completed before the crash). Because the flush task drains the value cache and the JMT pending queue together in a single `WriteBatch`, `state_cf` and `jmt_cf` are always consistent at the post-flush wave watermark — never half-flushed.
2. Walk the chain log forward from that wave to current head.
3. For each wave, replay its writes (extracted from the wave's transaction list and execution receipts) into the cache **and** rebuild the corresponding `TreeUpdateBatch` by re-running the JMT update against the now-current cache view.
4. Recompute state roots as we go to verify chain integrity.
5. Resume normal operation. The flush task will drain the rebuilt cache + pending queue on its next scheduled flush.

This is the standard write-ahead-log + write-back-cache crash recovery pattern. The chain log is durable; the cache and the pending JMT queue are optimization layers over RocksDB. Crash safety does not depend on either.

### Composition with PIP-2 and PIP-3

```
Layer                                 Where it lives                Win
─────────────────────────────────────────────────────────────────────────
Application write-back cache (PIP-4)  In-memory DashMap             Absorb write churn across waves
                                                                    Serve recent reads at memory speed
                                       │
                                       ▼
PIP-3 scheduler prefetch              RocksDB block cache           Pre-warm cold reads via batched MultiGet
                                       │
                                       ▼
PIP-2 clustered keys                  RocksDB sort order            Adjacent slots adjacent on disk
                                                                    Maximize block-cache reuse
                                       │
                                       ▼
RocksDB / disk                        SST files                     Authoritative on-disk state
```

A read for slot X in wave N:
1. Cache check (PIP-4): hit if X was read or written in the last 32 waves. Most reads in a steady-state workload land here.
2. PIP-3 prefetch: if X is in the wave's access list, the wave-level MultiGet has already pulled the surrounding SST blocks into RocksDB's block cache. The cold fall-through stays warm.
3. PIP-2 layout: even when we do hit RocksDB, the slot's neighbors are adjacent, so one block-read serves multiple sibling reads.

A write for slot X in wave N:
1. Cache write (PIP-4): O(1) hash insert. No disk IO.
2. Disk flush happens asynchronously, batched across many waves, with PIP-2 ensuring the writes hit a small set of SST regions instead of scattering.

## Rationale

### Why not synchronous flush?

Synchronous flush means each `sstore` blocks on RocksDB. Even with PIP-2 + PIP-3 reducing write amplification, the wall-clock cost per `sstore` is microseconds. At 50K `sstore` per wave and 500ms wave commit budget, synchronous flush alone could consume the entire budget on storage-heavy waves.

### Why DashMap and not a simpler concurrent map?

The wave executor processes transactions in parallel using access-list scheduling. DashMap's per-shard locking lets transactions touching disjoint slots run without contention. A single-lock `Mutex<HashMap>` would serialize the entire wave; an `RwLock<HashMap>` would serialize all writes. DashMap is the right primitive.

### Why a 32-wave warm window default?

Empirically observed in similar systems (Aptos's state cache, Solana's accounts db): write locality is high for the most recent ~10-50 waves and drops off rapidly after. A 32-wave window captures most re-write opportunities without bloating memory.

The window is configurable; high-write workloads may benefit from larger windows, memory-constrained validators from smaller.

### Why auto-tune on by default?

A fixed flush policy works well for one workload shape and poorly for others. Real validator workloads vary: bursty mempool periods, idle hours, recovery-from-stall replay loads. Auto-tune adapts to what the validator is actually seeing.

### Why is this v1, not v2?

Storage-bound workloads dominate real blockchain throughput. Without a write-back cache, sustained TPS will hit the RocksDB write ceiling well below the consensus and execution ceilings. The cache is what makes the rest of the architecture (Mysticeti's 500K-TPS-capable mempool, WASM execution at near-native speed) actually translate to user-visible TPS.

Deferring this to v2 means deferring the property that lets users feel the chain's design. The implementation cost is moderate (a few weeks of focused work). The user-visible impact is large. v1 it is.

## Backwards Compatibility

Application-layer change only. No protocol changes. No transaction-format changes. No consensus changes. Existing state and existing tests continue to work; only the path between `sstore` and RocksDB changes.

Validators upgrading to a PIP-4-enabled engine see immediately improved write throughput. Validators running pre-PIP-4 engines remain compatible with the network (consensus and state roots are unaffected; only local disk-write cadence differs).

## Reference Implementation

Staged across five PRs against `pyde-net/engine` on `execution-side`, each independently reviewable and trackable. The staging keeps each PR around the ~500-LoC focused-PR target the rest of β.1 follows:

| | Branch | Scope |
|---|---|---|
| PR-5a | `feat/state-cache-primitive` | `CacheStore` over `DashMap<SlotHash, CacheEntry>`, `EntryState` machine, `JmtPendingQueue`, unit tests — primitives only, no read/write-path wiring |
| PR-5b | `feat/state-cache-read-path` | `StateStore::read_slot` consults cache first; cache miss falls through to existing `state_cf` path (PR-3); cache-fill on read |
| PR-5c | `feat/state-cache-write-path` | `StateCommitter::commit` writes slot values to cache (Dirty) + enqueues the wave's `TreeUpdateBatch` on `JmtPendingQueue`; in-memory state-root computation; **no RocksDB writes per wave** |
| PR-5d | `feat/state-cache-flush-task` | Background flush task, three-signal policy, auto-tune, atomic cross-CF `WriteBatch` draining cache + queue, metrics |
| PR-5e | `feat/state-cache-crash-recovery` | Chain-log replay path; `kill -9` mid-wave integration test |

A successful overall implementation produces:
- DashMap-backed value cache with the entry shape above.
- Mutex-guarded `JmtPendingQueue` holding per-wave `TreeUpdateBatch` records.
- Read / write paths integrated into the state crate's public API (no changes to call sites; `read_slot` and `StateCommitter::commit` keep their existing signatures).
- Background flush task with the three-signal policy and auto-tune, draining both the value cache and the JMT pending queue through a single cross-CF `WriteBatch`.
- Crash-recovery replay code path (exercised via integration test simulating a kill -9 mid-wave).
- Metrics: cache hit rate, dirty count, JMT-queue depth, flush wall-clock, flush batch size.
- Property tests for correctness under simulated crashes.

## Test Plan

- Unit: per-shard correctness under concurrent reads + writes.
- Integration: end-to-end wave processing with simulated state churn; verify state roots match a reference (cache-disabled) implementation.
- Crash recovery: kill -9 the engine mid-wave; verify on restart that replay reconstructs state correctly and that the resulting state_root matches what the wave would have produced cleanly.
- Performance: measure write throughput before and after PIP-4 on a storage-heavy workload (token transfers, AMM swaps). Expected: 5-15x improvement in sustained-write TPS on workloads with locality.
- Memory: measure resident set under sustained load; confirm warm-window cap holds.

## Security Considerations

- **State integrity**: the cache is consulted on every read and every JMT update. A bug here would produce wrong state roots and a chain halt. Mitigations: extensive property tests, cache-disabled mode for differential testing, kill-9 crash-recovery integration tests.
- **Memory exhaustion**: an adversary issuing many writes to unique slots could grow the cache. Mitigated by the warm-window flush trigger and by the chain-level gas accounting that caps writes per wave.
- **Concurrent execution safety**: DashMap is well-tested; the per-entry state machine (Clean/Dirty/Pending) is invariant-checked in tests.

## References

- [PIP-2: Clustered state keys](./pip-0002-clustered-state-keys.md)
- [PIP-3: Scheduler-level state prefetch](./pip-0003-scheduler-prefetch.md)
- [Chapter 4: State Model](../pyde-book/src/chapters/04-state-model.md)
- [Chapter 3: Execution Layer](../pyde-book/src/chapters/03-virtual-machine.md)
