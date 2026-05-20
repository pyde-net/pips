---
pip: 3
title: Scheduler-level state prefetch
author: zarah <zarah.zrh2@gmail.com> (@zarah-s)
type: Standards Track
status: Draft
created: 2026-05-20
requires: 2
---

## Abstract

Add a *prefetch* step to the block executor: before each
execution wave runs, collect every storage key the wave's
transactions will touch (across all contracts and all txs in
the wave), and issue *one* RocksDB `MultiGet` call for the
whole list. Then run the wave. Each `Sload` issued by a
transaction in the wave hits the warm cache instead of going
to disk.

PIP-2 (clustered keys) makes keys for the same contract land
adjacent in RocksDB's sorted keyspace. RocksDB's `MultiGet`
internally sorts the input list, batches reads per SST file,
and parallelises I/O across files. So one wave-level
`MultiGet` over a key list of size N produces one bloom-
filter check per SST file plus one I/O per distinct block —
not N separate I/Os.

Together: cold-cache state reads drop from "N disk seeks per
wave" to "one I/O per distinct SST block per wave". Warm
reads are unchanged.

## Motivation

PIP-2's clustered key layout puts all of contract A's state
into a contiguous region of RocksDB. By itself this gives
real (but modest) wins via block-cache reuse: reading slot
`A.x` warms the block; the next read of slot `A.y` is warm.

The bigger win is **batched prefetch**. Every transaction
that arrives at the block executor carries an `access_list:
Vec<AccessEntry>` declaring the storage slots it will touch
(`engine/crates/tx/src/types.rs`). The scheduler already
groups transactions into parallel waves based on these
lists. With PIP-2's layout, the scheduler can do one more
thing: before running a wave, issue a *batched* read for
every slot the wave's transactions will touch, grouped by
contract.

The intent is to replace this access pattern:

```text
Wave runs ─► Tx1 starts ─► Sload(slot a) [cold I/O]
                       ─► Sload(slot b) [cold I/O]
                       ─► Sload(slot c) [cold I/O]
            Tx2 starts ─► Sload(slot d) [cold I/O]
                       ─► Sload(slot e) [cold I/O]
```

with this one:

```text
MultiGet([a, b, c, d, e])   [RocksDB sorts internally → keys for the
                             same contract land adjacent → 1 I/O per
                             distinct SST block; per-block I/Os run
                             in parallel inside RocksDB]
─► Wave runs ─► Tx1 starts ─► Sload(slot a) [warm]
                          ─► Sload(slot b) [warm]
                          ─► Sload(slot c) [warm]
               Tx2 starts ─► Sload(slot d) [warm]
                          ─► Sload(slot e) [warm]
```

For a token transfer block (the common case), this means
~one cold I/O per token-contract SST block instead of 2N cold
I/Os for N transfers. The exact win depends on workload
locality; the upper bound is "all reads in a wave become
warm."

## Specification

### Where the prefetch hooks in

The prefetch lives in the block executor (`engine/crates/tx/
src/pipeline.rs`), between wave scheduling and wave dispatch.
The control flow per wave:

```text
1. Scheduler produces wave W (a set of txs with no
   pairwise conflict on access-list slots).
2. Executor calls prefetch_wave(W) → blocks briefly
   until prefetches return.
3. Executor dispatches each tx in W to a worker thread.
4. Workers run; each Sload hits warm cache.
5. Wave commits; move to next wave.
```

Step 2 is the new step.

### `prefetch_wave` algorithm

```rust
// engine/crates/state/src/prefetch.rs (new module)

use crate::access::StateAccess;

/// Prefetch every slot a wave's transactions will read or
/// write. Returns when RocksDB has pulled the targeted SST
/// blocks into the block cache.
pub fn prefetch_wave(
    state: &dyn StateAccess,
    access_lists: &[&AccessList],
) -> Result<(), PrefetchError> {
    // 1. Collect every key the wave touches.
    let mut keys: Vec<H256> = Vec::new();
    for al in access_lists {
        for entry in al.entries() {
            keys.extend_from_slice(&entry.reads);
            // Writes: prefetch too — the SSTORE cost model
            // (Chapter 4.4 of the Otigen book) reads the
            // prior value, so warm reads help writes too.
            keys.extend_from_slice(&entry.writes);
        }
    }

    // 2. Dedupe. Two txs in the same wave never write the
    //    same slot (the scheduler enforces this) but they
    //    can read-share, and a key can appear in both reads
    //    and writes of the same tx.
    keys.sort_unstable();
    keys.dedup();

    // 3. One MultiGet for the whole wave.
    //    RocksDB sorts internally, batches per SST file,
    //    parallelises across files. That's the level the
    //    parallelism wants to live at.
    state.multi_get_for_warm(&keys)
}
```

The `multi_get_for_warm` API is added to `StateAccess`:

```rust
trait StateAccess {
    // ... existing methods ...

    /// Read the given keys and discard the values; the
    /// effect is to warm the block cache. Returns once
    /// every key has been resolved (either from cache or
    /// from disk).
    fn multi_get_for_warm(
        &self,
        keys: &[H256],
    ) -> Result<(), StateError>;
}
```

The implementation in `JmtRocksStore` calls
`rocksdb::DB::multi_get_cf` with the storage column family.
That's the entire body of the function.

### Synchronous vs fire-and-forget

The reference implementation makes `prefetch_wave` *block*
until every contract's MultiGet returns. Total wave
latency is then `prefetch_time + execution_time` vs the
no-prefetch baseline of `execution_time_with_cold_reads`.

The reason: RocksDB's block cache is best populated *before*
the execution thread asks for the key. If we return early
and the execution thread races ahead of the I/O, it issues
its own cold read, defeats the prefetch, and we've spent
the prefetch cost for nothing.

A future enhancement could overlap prefetch of wave N+1
with execution of wave N, getting the wins of both. The
v1 design is synchronous to keep the interaction with the
existing wave-execution code simple.

### Interaction with Block-STM speculation

When a transaction has an *incomplete* access list (the
runtime will discover slots dynamically — covered in
Chapter 15.3 of the Otigen book), the prefetch can only
fetch the *statically known* slots. The dynamic slots are
discovered during execution and read cold.

This is fine — prefetch is a best-effort optimization. The
fraction of slots that are statically known is exactly the
fraction that benefit. The dynamic slots fall through to
normal cold reads. No correctness implication.

### JMT internal-node warming

The JMT stores its internal tree nodes as RocksDB KV pairs.
A leaf read requires traversing log₁₆(N) internal nodes
first. With PIP-2's clustering, internal nodes that share a
common address prefix in their JMT path are themselves
clustered in RocksDB.

When the MultiGet prefetches a leaf, it must traverse the
internal path to get there. That traversal warms the
internal nodes along the way. Subsequent leaf reads for the
same contract reuse the warm internal path.

So we don't need to explicitly prefetch internal nodes —
the act of multi-getting leaves brings the internal nodes
along.

### Write-path prefetch

Writes (`Sstore`) need to read the previous value to
compute the SSTORE cost (fresh-slot vs modify vs no-op,
per Chapter 4.4 of the Otigen book). So writes also
benefit from prefetch — the prior-value read happens
against the warm cache.

The reference implementation prefetches both reads and
writes from each access list entry; no distinction at the
prefetch layer.

### Activation

Pre-mainnet. PIP-3 is purely additive to the existing
runtime — it adds a step before wave dispatch. No on-chain
state changes; no consensus changes; no wire-format
changes. Lands in a single PR against `engine/crates/state/`
and `engine/crates/tx/`.

If for some reason PIP-3 had to land post-mainnet, it
would still require no hard fork — it's an unobservable
performance optimization on the node side. Nodes that
prefetch and nodes that don't produce the same state root
and the same receipts; they just spend different amounts
of wall time getting there.

## Rationale

### Why scheduler-level rather than PVM-level

A PVM-level alternative would have the VM emit a "prefetch
hint" instruction at the start of each function, listing
slots it will touch. This was rejected:

- The VM already has the access list at runtime via the
  PVM's `allowed_storage_keys` check; emitting it again as
  a hint instruction would be redundant.
- Per-function granularity over-issues prefetches across
  many small functions. Per-wave is the right grain.
- Cross-tx batching is impossible from inside one VM
  instance.

A RocksDB-level alternative would be an LSM-aware
prefetch policy ("when reading key prefix X, speculatively
load nearby keys"). RocksDB doesn't support this natively;
adding it via the EventListener API is fragile and
requires modifying the upstream dep. Scheduler-level is
simpler and gives the application full control.

### Why one wave-level MultiGet (not per-contract, not per-tx)

An earlier draft of this PIP grouped the wave's keys by
contract address and issued one `MultiGet` per contract,
fanned out across a thread pool. That design was rejected
in review: it duplicates work RocksDB already does
internally.

RocksDB's `multi_get_cf`:

- **Sorts the input keys** before reading. With PIP-2's
  clustering, keys for the same contract are then adjacent
  in the sorted batch.
- **Checks each SST file's bloom filter once per file**, not
  once per key. A wave touching 200 keys across 3 SST files
  does 3 bloom checks, not 200.
- **Parallelises I/O across SST files** internally. The
  application doesn't need its own thread pool.
- **Pins blocks across hits.** Two keys landing in the same
  block trigger one read; both are served from it.

So feeding the entire wave's key list to a single
`MultiGet` call gives us all the locality and parallelism
benefits with one syscall and no application-level thread
management. Per-contract grouping would force N separate
syscalls and an external thread pool to coordinate them —
strictly more overhead.

Per-slot (one prefetch per slot) doesn't batch at all;
you'd issue N `MultiGet`s where one would do. Same
problem, worse.

### Why synchronous prefetch (v1)

Asynchronous prefetch ("issue prefetch for wave N+1 while
executing wave N") would extract more parallelism, but it
requires either a separate I/O thread pool or a coroutine-
based execution model. The v1 design avoids this
complexity. A future PIP can add async prefetch as an
optimization on top.

### Why MultiGet over Iterator

RocksDB has two relevant APIs:

- **`Iterator::seek(prefix)` + scan**: pulls SST blocks
  containing the prefix into the cache.
- **`MultiGet(keys)`**: issues N point lookups, batched.

MultiGet is the right tool here. The access list gives us
*exact* keys, not a range. MultiGet targets only those
keys; it doesn't scan unrelated state in the same SST
block. For state-sync ("give me all of contract X"),
Iterator is the right choice — but that's a different
feature, not what prefetch needs.

## Backwards compatibility

None — purely additive. The prefetch step is invisible to
contracts, to clients, and to the wire format. Removing
the prefetch step is also free; the only effect is that
cold reads become cold reads.

## Security considerations

### No observable change

Prefetch only affects the *timing* of state reads, not
their content. The block's resulting state root, receipts,
and event logs are identical whether prefetch runs or not.
Two nodes with different prefetch policies still produce
the same chain.

### Cache-pressure DOS

A malicious transaction could in principle declare a
huge access list (many slots across many contracts) to
force the executor to prefetch a large working set,
filling the block cache and evicting other workloads'
data.

Mitigation: gas. The access list is part of the
transaction's signed payload; declaring it costs gas
proportional to its size (`crates/tx/src/fee.rs`). A
malicious user pays per slot they declare. Combined with
the block gas limit, this caps the per-block prefetch
budget.

Additionally, the PVM enforces the access list — any slot
not declared cannot be read. So a transaction cannot
*both* declare a huge prefetch AND actually use a small
working set; it'd revert with `AccessListViolation`. The
declared-but-unused case is precisely the attack vector
gas pricing already handles.

### Block-cache poisoning

A malicious validator could in principle pre-populate
the block cache with stale state to confuse downstream
reads. This is not a new attack surface — it exists
without prefetch — and is mitigated by RocksDB's existing
consistency model (snapshot reads see a consistent
version of the database).

## Reference implementation

To be drafted as a single PR against `engine/`:

- New module `engine/crates/state/src/prefetch.rs`
  containing `prefetch_wave` and the `multi_get_for_warm`
  trait method on `StateAccess`.
- Implementation of `multi_get_for_warm` in
  `engine/crates/state/src/jmt_store.rs` using
  `rocksdb::DB::multi_get_cf`.
- Memory implementation in
  `engine/crates/state/src/memory_jmt.rs` as a no-op
  (in-memory state is already "warm").
- Wire-up in `engine/crates/tx/src/pipeline.rs`: call
  `prefetch_wave(state, &access_lists)` before
  dispatching each wave.
- Benchmark in `engine/crates/state/benches/` measuring
  wall-clock cold-cache wave execution with vs without
  prefetch.

### Bench plan

Three benchmarks, each comparing PIP-2-only vs PIP-2+PIP-3:

1. **Synthetic single-contract token transfer block.**
   1000-holder token; 100 transfers between random pairs;
   cold cache at block start.
   - Expected (PIP-2-only): ~200 leaf reads + ~200 internal-
     node reads = ~400 cold I/Os.
   - Expected (PIP-2+PIP-3): 1 prefetch MultiGet (200 keys),
     surrounding internal nodes warmed in the process =
     ~1 cold I/O + a few warm internal-node reads.
   - Target: 50x+ cold-read reduction.

2. **Realistic mixed-contract block.** 10 contracts touched
   in one block; ~20 txs per contract; mix of cold and warm.
   - Expected: prefetch wins on the cold contracts (3-10x);
     warm contracts unaffected.
   - Target: overall block wall-clock improvement of 2-3x
     on cold blocks; ≤5% overhead on warm blocks.

3. **Adversarial large access list.** Tx declares 1000-slot
   access list but only uses 1. Verify (a) prefetch doesn't
   crash, (b) gas pricing eats the cost, (c) the AccessList
   Violation check still fires on real misuse.
   - Target: no regression; gas charged as expected.

### Acceptance gate

The PIP is accepted if and only if:

- Bench 1 shows ≥10x cold-read reduction.
- Bench 2 shows ≥2x cold-block wall-clock improvement.
- Bench 2 shows ≤5% regression on warm blocks.
- Bench 3 shows no security regression.

If any of these fails, PIP-3 is withdrawn rather than
accepted.

## Open questions

1. **Prefetch budget cap.** Should there be a hard cap on
   the number of slots prefetched per wave (or per
   contract), to bound the worst-case block cache
   eviction? Probably yes, but the cap value depends on
   the cache size and workload. Leaving the detail to the
   reference implementation; the cap value can be tuned
   without a PIP amendment.

2. **Async prefetch (v2).** Should v1 ship synchronous
   prefetch only, with async prefetch as a follow-up PIP?
   Yes — the v1 design is small and self-contained; async
   adds a thread pool and coroutine-style scheduling that
   warrants its own design pass.

3. **Static access-list inference timing.** The compiler-
   side access-list inference (Otigen book Ch 15.1) is
   designed-but-not-yet-shipped. PIP-3 works against
   whatever access list the tx carries — submitter-
   declared, simulator-discovered, or eventually compiler-
   inferred. PIP-3 has no dependency on the inference
   landing; it just benefits more once it does.

4. **Cross-block prefetch.** Should the executor prefetch
   into the *next* block while the current block executes?
   Out of scope for v1; would require pipeline-aware
   scheduling at a layer above the block executor.

## Implementation ordering with PIP-2

PIP-2 must land first. Without clustered keys, PIP-3's
MultiGet still works correctly, but the locality benefit
disappears — each key is in its own SST block, and
MultiGet issues N I/Os instead of one. PIP-3 alone (on
the current hashed layout) would still be additive but
~Nx less effective than PIP-2+PIP-3.

The reference implementation for PIP-3 can be drafted in
parallel with PIP-2, but it should not merge until PIP-2
is merged and the layout change is in effect on whatever
network the bench is running against.
