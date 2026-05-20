---
pip: 2
title: Clustered state keys (address-prefix layout)
author: zarah <zarah.zrh2@gmail.com> (@zarah-s)
type: Standards Track
status: Draft
created: 2026-05-20
---

## Abstract

Change Pyde's storage-key derivation so that every key for a given
contract sorts together in the underlying KV store. Replace the
current `Poseidon2(contract_address || discriminator || …)` form,
which scatters one contract's slots randomly across the keyspace,
with a `(contract_address_prefix || Poseidon2(discriminator || …))`
form, where the high 16 bytes of every storage key are the
contract's own address. This is a one-time pre-mainnet layout
change.

The result: one RocksDB iterator with a 16-byte prefix returns the
entire contract's state in one ordered range scan, instead of
needing N point lookups scattered across the SST tree. Cold-cache
multi-slot reads, access-list-driven prefetch, state-sync, and
indexer/explorer "give me all of contract X" queries all
benefit.

## Motivation

Pyde uses JMT (Jellyfish Merkle Tree) backed by RocksDB to store
contract state. Each storage field gets a slot index assigned at
compile time; the runtime derives the slot's storage key by
hashing `(contract_address, discriminator_byte, slot_index)` (and
optionally a map key) through Poseidon2.

Because the contract address is mixed _into_ the hash, two slots
of the same contract end up at unrelated points in the
256-bit keyspace. Reading slot 0 and slot 1 of contract `A` is
two independent disk seeks (in cold cache), even though
conceptually they're "the same contract's data".

This matters for several access patterns the chain already
supports first-class:

1. **Access-list-driven prefetch.** The scheduler knows in
   advance which slots a transaction will touch (Ch 15 of the
   Otigen book, the access-list mechanism in
   `engine/crates/tx/src/types.rs`). On the current layout,
   that knowledge only enables conflict detection — it doesn't
   help with I/O batching, because the slots live in unrelated
   parts of RocksDB. With clustered keys, the scheduler can
   issue _one_ range scan keyed on the contract's address and
   pull the SST block containing all touched slots into cache
   before the transaction runs.

2. **Cold-cache multi-slot reads.** A typical transfer touches
   `balances[from]` and `balances[to]` — both in the same
   contract. Today these are two cold-cache point lookups (when
   the contract is cold). With clustered keys, one block read
   serves both. The expected win is 2–10× on cold reads at
   typical workload patterns; warm-cache reads are unchanged.

3. **State sync.** Streaming a contract's complete state to a
   joining peer becomes one ordered range scan instead of a
   per-slot reconstruction.

4. **Explorers and indexers.** "Show me the full state of
   contract X" becomes one range scan instead of N selector-
   driven point queries.

The change has no effect on the JMT's safety properties, the
parallel-execution model, sparse-map semantics, or the access-
list enforcement at the PVM layer. It is a layout optimization
on the _physical_ placement of keys in RocksDB.

## Specification

### New key-derivation rule

For every storage key the runtime computes, the layout is:

```text
                    32 bytes total
┌──────────────────────────────┬──────────────────────────────┐
│ contract_address_prefix      │ Poseidon2(discriminator      │
│ (high 16 bytes of the        │            || slot_index     │
│  contract's 32-byte address) │            || map_key?)      │
│                              │ truncated to low 16 bytes    │
└──────────────────────────────┴──────────────────────────────┘
```

Concretely, the existing functions in
`engine/crates/state/src/keys.rs` change as follows.

**Account-metadata keys** (balance, nonce, code, code_hash,
aot_code):

```rust
// Before:
//     key = Poseidon2(address || discriminator_byte)
// After:
fn account_meta_key(address: &Address, discriminator: u8) -> H256 {
    let hashed = poseidon2_hash(&[discriminator]).to_bytes();
    let mut out = [0u8; 32];
    out[..16].copy_from_slice(&address[..16]);
    out[16..].copy_from_slice(&hashed[..16]);
    H256::from(out)
}
```

**Storage-slot keys**:

```rust
// Before:
//     key = Poseidon2(contract || 0x04 || slot_index_le)
// After:
fn storage_slot_key(contract: &Address, slot: u64) -> H256 {
    let mut input = Vec::with_capacity(9);
    input.push(discriminator::STORAGE_SLOT);
    input.extend_from_slice(&slot.to_le_bytes());
    let hashed = poseidon2_hash(&input).to_bytes();

    let mut out = [0u8; 32];
    out[..16].copy_from_slice(&contract[..16]);
    out[16..].copy_from_slice(&hashed[..16]);
    H256::from(out)
}
```

**Map-entry keys** (single-level):

```rust
// Before:
//     key = Poseidon2(Poseidon2(contract || 0x04 || slot)
//                     || 0x05 || map_key_bytes)
// After:
fn map_entry_key(contract: &Address, slot: u64, map_key: &[u8]) -> H256 {
    // Hash the intra-contract path only (no address):
    let mut slot_input = Vec::with_capacity(9);
    slot_input.push(discriminator::STORAGE_SLOT);
    slot_input.extend_from_slice(&slot.to_le_bytes());
    let slot_hash = poseidon2_hash(&slot_input);

    let mut entry_input = Vec::with_capacity(33 + map_key.len());
    entry_input.extend_from_slice(slot_hash.as_slice());
    entry_input.push(discriminator::MAP_ENTRY);
    entry_input.extend_from_slice(map_key);
    let entry_hash = poseidon2_hash(&entry_input).to_bytes();

    let mut out = [0u8; 32];
    out[..16].copy_from_slice(&contract[..16]);
    out[16..].copy_from_slice(&entry_hash[..16]);
    H256::from(out)
}
```

**Nested map-entry keys**: analogous, chaining two `Poseidon2`
calls on the intra-contract portion before the address-prefix
splice.

**Validator and global system keys** (no contract address):
keep the existing form. Their address-equivalent prefix is the
discriminator byte, so they cluster together naturally without
the splice.

### Key layout summary

After the change, _every_ storage key for contract `A` starts
with `address_of_A[..16]`. RocksDB sorts keys lexicographically;
all of `A`'s state therefore lives in a contiguous range of the
SST tree. A RocksDB iterator seeked to that prefix returns
exactly `A`'s state in slot order, with no other contract's
data interleaved.

### JMT impact

The `jmt` crate's `KeyHash` type is `[u8; 32]`. The new layout
remains 32 bytes total, so no JMT-side changes are required.
JMT internal nodes will see longer shared prefixes (every leaf
under contract `A` shares a 16-byte prefix), which the
radix-16 path-compressed JMT collapses into a small number of
high-up nodes. Tree depth is unchanged; the path-compressed
top of the tree gets denser.

### PVM impact

`Sload` and `Sstore` opcodes already operate on 256-bit (32-
byte) storage keys via `H256`. No opcode or instruction-format
change.

### Activation

This is a pre-mainnet layout change. It does not need a hard
fork because no mainnet state exists yet — it lands in a
single PR against `engine/`, regenerates any test fixtures
that hardcode key bytes, and ships as the v1 layout.

If for any reason the change must instead be deployed post-
mainnet, it requires a hard fork with a state-migration step
that re-derives every existing storage key under the new
layout (effectively a copy-and-rehash of the entire state
tree). This is operationally expensive and is the case for
landing the change pre-mainnet.

## Rationale

### Why 16-byte address prefix + 16-byte intra-contract hash

The 32-byte key budget splits naturally:

- **16 bytes of address prefix** is 128 bits of clustering
  identity. To force a different contract's state to sort
  next to yours, an attacker would need to grind a contract
  address whose high 16 bytes match a target — a `2^128`
  operation. Computationally infeasible. 16 bytes is enough
  clustering to put thousands of contracts' SST blocks in
  distinct regions of disk.

- **16 bytes of intra-contract hash** is 128 bits of collision
  space within a single contract. With Poseidon2 as the hash,
  the expected first collision in a contract with `n` slots
  is at `n ≈ 2^64`. Real contracts have at most ~10^6 slots
  (a token with a million unique address holders); the
  collision probability is `2^-104`. Negligible.

The split is symmetric and gives both halves more than enough
security margin.

### Why not full 64-byte keys

A 64-byte key (32 bytes of address + 32 bytes of hash) would
preserve full hash widths on both sides, but requires either:

- modifying the `jmt` external dependency to use a wider key
  type (mechanically invasive, touches a crate Pyde doesn't
  control), or
- introducing a two-layer system where RocksDB uses 64-byte
  keys but JMT hashes the (addr, intra-hash) pair to 32 bytes
  for Merkle commitment.

The first is operationally painful (forks an upstream
dependency). The second works but adds an extra hash per slot
access and a second indexing layer, with marginal benefit over
the 16+16 split. Rejected.

### Why not per-contract JMT subtree

A "tree of contracts, each contract a JMT subtree" design is
semantically clean: each contract's state has its own Merkle
root, and the top-level state root is a hash of the per-
contract roots. State sync and proofs of "all of contract X"
become natural.

But it's a substantially larger change — different state
commitment, different proof shape, different `Sload`/`Sstore`
internals. It's worth considering for a future PIP if the
cost-of-Merkle-proof per contract becomes a bottleneck, but
the address-prefix lift achieves most of the practical I/O
benefit with a fraction of the engineering surface.

### Why not change the JMT, just the RocksDB layout

JMT's storage layout is part of how it commits to state; the
key inside JMT must match the key the prover commits to.
Changing only the RocksDB layout (without changing the JMT
key) would mean RocksDB's clustering doesn't help, because
JMT lookups produce a tree path keyed by the hash, and RocksDB
serves the JMT nodes (not the slot keys directly). The two
must move together. This PIP changes the key both pass
through.

### Why pre-mainnet

A post-mainnet version of this PIP would require a state
migration: re-hash every storage key under the new layout,
re-merkleize the state tree, and have nodes verify the new
root against a deterministic transition. This is
operationally expensive and complicates state-sync during
the transition window. Pre-mainnet, the change is a single
PR.

## Backwards compatibility

None — this is a pre-mainnet change. No on-chain state
references the old key layout because mainnet hasn't shipped.

Devnets and testnets running under the old layout would need
their state wiped. The compiler-emitted slot indices and the
ABI are unchanged; contracts compiled before the change
continue to work as-is after the change.

Tests in `engine/crates/state/src/keys.rs` that hardcode
expected key bytes will fail until updated. These are
deterministic regenerations.

## Security considerations

### Collision resistance

The intra-contract hash space is 128 bits. Within a single
contract, the probability of a collision between any two
slot keys (a primitive field vs a map entry, two map entries
under different keys, etc.) is `< 2^-100` at realistic
contract sizes. The discriminator-byte system that protects
namespace separation today is preserved (the discriminator
is part of the hash input).

### Address grinding

A user could grind their contract address to share a high-16-
byte prefix with another contract. This is a `2^128`
operation — computationally infeasible. Even if it were
feasible, the only effect would be that the two contracts'
SST blocks live in the same region of disk, which has no
correctness implication (their intra-contract hashes still
keep their keys distinct).

### Adversarial map keys

A user submitting a transaction that writes to a map can
choose the map key. With the new layout, the map key still
goes through Poseidon2 alongside the discriminator and slot
index, so the resulting intra-contract hash is collision-
resistant. Slot-vs-map collisions within a contract are
prevented by the discriminator byte (`0x00`/`0x01`/... for
primitive slots vs `0x05` for map entries) appearing in the
hash input. Unchanged from today.

### Post-quantum properties

Poseidon2 is the hash function, both before and after. No
change to the post-quantum security argument.

### Proof size

Merkle proofs grow by a single nibble step at worst, because
the JMT depth is unchanged (path compression collapses the
shared address prefix into a small number of nodes). For ZK
proofs of individual slots, the in-circuit cost is
essentially unchanged — Poseidon2 hash count along the path
is the same.

## Reference implementation

To be drafted as a single PR against `engine/`:

- `engine/crates/state/src/keys.rs` — update every key-
  derivation function to use the new layout.
- `engine/crates/state/src/keys.rs::tests` — regenerate the
  hardcoded key bytes in unit tests.
- `engine/crates/state/src/jmt_store.rs` — sanity-check that
  the JMT integration still works against the new key shape
  (it should, since the key type didn't change).
- Add a benchmark in `engine/crates/state/benches/` that
  measures cold-cache N-slot read for one contract under
  both layouts.

Acceptance is gated on the benchmark confirming a measurable
win (target: ≥2× on cold reads of 4+ slots from one contract;
no measurable regression on warm cache or single-slot reads).
If the benchmark fails to show a material win, this PIP is
withdrawn rather than accepted — the design is clean enough
to ship but only worth shipping if the practical effect
matches the model.

## Open questions

1. **Bench shape.** What workload should be the primary
   benchmark? Candidates: replaying a representative
   token-transfer block; a synthetic "N transfers between
   N distinct address pairs" microbench; a state-sync
   replay of a real testnet block. The PR should include
   at least the synthetic microbench and one realistic
   replay.

2. **State-sync RPC.** The clustered layout naturally
   supports a "give me all of contract X" RPC method.
   Should this be added in the same PIP or a follow-up?
   Lean toward follow-up — the layout change is the
   prerequisite; the RPC surface is its own design.

3. **Explorer integration.** Should explorer tooling
   immediately consume the new "all of contract X" form
   via RocksDB iterators (where the explorer has direct
   storage access) or wait for the RPC method? Follow-up.
