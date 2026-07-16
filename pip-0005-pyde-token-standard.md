---
pip: 5
title: PTS — the Pyde Token Standard (pts-f/1, pts-n/1)
author: zarah (@zarah-s)
type: Standards Track
status: Draft
created: 2026-07-16
---

## Abstract

PTS defines Pyde's native standards for fungible (`pts-f/1`) and
non-fungible (`pts-n/1`) tokens as **manifest types**: a token is not
code an author writes but configuration (`type = "token"` +
`standard = "..."` in `otigen.toml`) that the otigen toolchain
compiles from one audited canonical implementation. Token manifests
are config-only — declaring functions, state, events, or shipping
source is a build error — so every deployed token is verifiable by
rebuilding its manifest and comparing bytes, and misimplementation is
unrepresentable. The generated surface eliminates the documented
failure classes of inherited token models: standing-approval drains
(allowances are delta-only with mandatory, TTL-capped expiry),
tokens stranded in unaware contracts (settle-then-notify
`transfer_call` with a mandatory acknowledgement), callback
reentrancy (no sender hooks; notification fires only after full
settlement, under the engine's per-function guard), and
retroactively-enabled issuer powers (capabilities are
manifest-declared, artifact-visible, never addable post-deploy). No
consensus change is required: PTS tokens deploy as ordinary
`ContractType::Contract` artifacts; the chain remains token-ignorant.

## Motivation

Pyde currently ships token examples named after another chain's
proposal registry (ERC-20/ERC-721 shapes). Beyond the naming, the
inherited model carries structural, measured costs, none of which are
bugs in individual tokens:

- **Standing approvals**: ~$417–493M stolen via token approvals since
  2020 (BadgerDAO ~$120M alone); roughly 60% of live approvals on
  EVM chains are unlimited.
- **Off-chain approval signatures** (`permit`, Permit2): 56.7% of
  2024's token-phishing losses; single thefts up to $6.5M from one
  invisible signature.
- **No recipient handshake**: $103.79M measured (Oct 2025) in tokens
  permanently stuck at contract addresses across ~2,409 tokens.
- **Ambient transfer hooks** (ERC-777): $40M+ across Uniswap V1
  imBTC, Lendf.me ($25.2M), Cream ($18.8M); the reference library
  deleted its implementation.
- **Under-specification**: inconsistent `transfer` returns forced a
  permanent wrapper-library tax; optional metadata forced the
  off-chain token-list detour; unspecified vault rounding (ERC-4626)
  shipped a recurring exploit class ($30M+).

Every ledger designed after Ethereum converged on one audited
implementation instead of per-token code (Solana's token program,
Cosmos `x/bank`, Aptos Fungible Asset, Polkadot `pallet-assets`);
both contract-level standards that competed against a native module
(CW20, PSP22) lost. Pyde can obtain the same property **without
engine changes** because the platform already carries the ABI inside
each artifact (`pyde.abi` custom section), derives storage slots from
a declared schema, and drives four language toolchains from one
manifest. The standard therefore lives at the toolchain layer, where
it can be generated, validated, and mechanically conformance-checked.

## Specification

### 1. Kind system

`[contract]` gains one field, `standard`:

| `type =`     | `standard =`            | Result                                |
|--------------|-------------------------|---------------------------------------|
| `"contract"` | absent (presence = build error) | ordinary hand-written contract |
| `"token"`    | `"pts-f/1"`             | generated fungible surface            |
| `"token"`    | `"pts-n/1"`             | generated non-fungible surface        |
| `"token"`    | missing / unknown       | build error listing known standards   |

Prose names are PTS-F / PTS-N; wire identifiers are lowercase
versioned strings. `standard` is a single scalar — a fused
fungible/non-fungible artifact is unrepresentable. On-chain, tokens
deploy as `ContractType::Contract` (tag `0x00`); no new
`ContractType` variant is introduced.

### 2. Manifest schema (pts-f/1)

```toml
[contract]
name     = "<chain-registered name>"   # ENS-style rules, registered at deploy
version  = "<semver>"
type     = "token"
standard = "pts-f/1"

[token]
name           = "<string>"     # 1–64 bytes UTF-8; control characters rejected
symbol         = "<string>"     # 1–12 chars; [A-Z0-9] recommended
decimals       = 9              # u8, ≤ 18; OMITTED → 9 (native parity)
initial_supply = "<u128>"       # smallest units; minted at init
initial_holder = "deployer"     # address | "deployer" (default)

[token.supply]
minter     = "deployer"         # address | "deployer" | "none" (mint code stripped)
manager    = "deployer"         # address | "deployer" | "none"
max_supply = "<u128>"           # optional; omitted = uncapped (reported as 0)
burnable   = true               # default true

[token.extensions]              # every key optional; off = code absent
freeze       = false
pause        = false
registration = "open"           # "open" (default) | "required"
metadata_uri = ""               # non-empty enables the extension

[token.hooks]
transfer_call = true            # default true
```

**Validation (build-time, otigen):** `standard` required for tokens
and forbidden otherwise; any `[functions.*]`, `[state]`, `[events]`
section or source directory on `type = "token"` is an error;
cross-standard keys are errors naming the key and owning standard
(`decimals` is pts-f-only; per-id metadata keys are pts-n-only);
symbol/name length and charset enforced; `decimals ≤ 18`; supplies
must parse as u128 with `initial_supply ≤ max_supply` when capped;
role values must be `"deployer"`, `"none"`, or a well-formed 32-byte
address. `"deployer"` resolves at deploy time to the deploying
account. `"none"` strips the corresponding code from the artifact.

### 3. Constants

| Constant | Value | Meaning |
|---|---|---|
| `MAX_ALLOWANCE_TTL_WAVES` | `63_072_000` | ≈ 1 year at 500 ms/wave; hard cap on allowance lifetime |
| `MAX_BATCH` | `256` | max `owners` in `balance_of_batch` |
| `MAX_CALL_DATA` | `4_096` | max bytes of `data` in `transfer_call` |
| `ACK_TOKEN` | `u32::from_le_bytes(Blake3("pts/on_token_received/1")[0..4])` | required return of a fungible receiver |
| `ACK_NFT` | `u32::from_le_bytes(Blake3("pts/on_nft_received/1")[0..4])` | required return of an NFT receiver |
| `ROLE_MINTER` / `ROLE_MANAGER` / `ROLE_FREEZER` / `ROLE_PAUSER` | `Blake3("minter") / …("manager") / …("freezer") / …("pauser")` | 32-byte role identifiers carried by `RoleTransfer` |

### 4. Conventions (normative)

1. Amounts are `u128` smallest units. `decimals` is display-only.
2. Mutating functions return nothing; failure is a revert whose
   reason is exactly one canonical code from §7.
3. No generated function carries the `PAYABLE` attribute except
   `register()` (§9), which MUST be revert-free by construction.
4. Function names are exact `snake_case` as written below; dispatch
   is by name and no overloading exists.
5. Generated storage writes zero rather than deleting for balances
   and allowances (`sdelete` is never emitted on those paths).
6. **Commit-reveal neutrality:** no conformant function may depend on
   mempool visibility, same-wave ordering, or first-observer state
   races; behavior under a commit-reveal-wrapped submission MUST be
   identical to a plain submission.
7. Generated artifacts carry correct per-function access lists.

### 5. Fungible surface (pts-f/1) — views

All views carry the `VIEW` attribute.

```
fn standard() -> String                     // "pts-f/1"
fn name() -> String
fn symbol() -> String
fn decimals() -> u8
fn total_supply() -> u128
fn max_supply() -> u128                     // 0 = uncapped
fn balance_of(owner: Address) -> u128       // missing slot → 0
fn balance_of_batch(owners: Vec<Address>) -> Vec<u128>   // len > MAX_BATCH reverts
fn allowance(owner: Address, spender: Address) -> u128   // 0 once expired (lazy)
fn allowance_expiry(owner: Address, spender: Address) -> u64
fn token_info() -> TokenInfo   // Borsh struct {name, symbol, decimals,
                               //  total_supply, max_supply, minter,
                               //  extension_flags: u32}
fn minter() -> Address                      // ZERO = renounced / stripped
fn manager() -> Address
fn is_frozen(owner: Address) -> bool        // freeze extension only
fn is_paused() -> bool                      // pause extension only
fn is_registered(owner: Address) -> bool    // registration extension only
```

`extension_flags` bit assignments: bit 0 freeze, bit 1 pause, bit 2
registration-required, bit 3 metadata_uri, bit 4 transfer_call, bit 5
burnable. Bits 6–31 reserved.

### 6. Fungible surface (pts-f/1) — mutations

```
fn transfer(to: Address, amount: u128)
```
Debits `caller()`, credits `to`. Reverts `token:invalid_recipient`
if `to` is the zero address or the token's own address;
`token:insufficient_balance` on shortfall; `token:frozen` /
`token:paused` / `token:not_registered` per active extensions. MUST
NOT invoke any code on `to`.

```
fn transfer_call(to: Address, amount: u128, data: Vec<u8>) -> u32
```
Ordering is normative: (1) full balance settlement, (2) `Transfer`
emission, (3) `cross_call` of `on_token_received(operator, from,
amount, data)` on `to` where `operator = caller()` and `from =
caller()`. Reverts `token:bad_receiver` unless the callback returns
`ACK_TOKEN`. A recipient revert unwinds the entire operation.
`data.len() > MAX_CALL_DATA` reverts. `to` restrictions as
`transfer`.

```
fn transfer_from(from: Address, to: Address, amount: u128)
```
Spends a live allowance of `(from, caller())`: reverts
`token:allowance_expired` if `current_wave > expiry_wave`,
`token:insufficient_allowance` on shortfall; then behaves as
`transfer(from → to)`. Decrements the allowance; emits `Transfer`;
MUST NOT emit `Approval`. Operator attribution is derivable from the
enclosing transaction/receipt, not the event.

```
fn approve(spender: Address, amount: u128)
```
Compatibility form: sets the allowance to exactly `amount` with
`expiry_wave = current_wave + MAX_ALLOWANCE_TTL_WAVES`. Emits
`Approval`.

```
fn increase_allowance(spender: Address, amount: u128, expiry_wave: u64)
fn decrease_allowance(spender: Address, amount: u128)
fn revoke_allowance(spender: Address)
```
Delta semantics; `increase_allowance` reverts `token:overflow` on
u128 overflow and `token:invalid_expiry` if `expiry_wave ≤
current_wave` or `expiry_wave > current_wave +
MAX_ALLOWANCE_TTL_WAVES`. `decrease_allowance` floors at zero.
`revoke_allowance` writes `{0, 0}`. Each emits `Approval` with
absolute post-state.

```
fn set_allowance_exact(spender: Address, expected_remaining: u128,
                       new_remaining: u128, expiry_wave: u64)
```
Compare-and-set: reverts `token:allowance_changed` unless the current
*effective* remaining (0 if expired) equals `expected_remaining`.
Emits `Approval`. Note: under commit-reveal the check may fail at
reveal time if state moved after commit; integrators MUST handle this
revert.

```
fn mint(to: Address, amount: u128)          // minter only; token:not_minter
                                            // token:cap_exceeded above max_supply
fn burn(amount: u128)                       // requires burnable = true
fn burn_from(from: Address, amount: u128)   // spends allowance, then burns
fn set_minter(new: Address)                 // manager only
fn set_manager(new: Address)                // manager only
fn set_freezer(new: Address)                // manager only; freeze ext
fn set_paused(paused: bool)                 // pauser only; pause ext
fn freeze(account: Address, frozen: bool)   // freezer only; freeze ext
fn register()                               // registration ext; PAYABLE; idempotent
fn deregister()                             // returns the bond; zeroes the slot
```
`mint`/`burn` are the ONLY writers of `total_supply` and emit
`Transfer` with the zero-address sentinel (`from = ZERO` for mint,
`to = ZERO` for burn). Role setters emit `RoleTransfer`; the zero
address renounces provably.

### 7. Canonical revert codes

`token:insufficient_balance`, `token:insufficient_allowance`,
`token:allowance_expired`, `token:allowance_changed`,
`token:invalid_expiry`, `token:invalid_recipient`, `token:overflow`,
`token:not_minter`, `token:not_manager`, `token:not_freezer`,
`token:not_pauser`, `token:cap_exceeded`, `token:not_burnable`,
`token:frozen`, `token:paused`, `token:not_registered`,
`token:bad_receiver`, `token:batch_too_large`,
`token:data_too_large`; pts-n/1 additionally: `token:nonexistent`,
`token:not_owner`, `token:not_authorized`, `token:invalid_operator`.
Conformance vectors pin the exact bytes.

### 8. Storage schema (pts-f/1)

Typed-storage fields (slots host-derived per the HOST_FN_ABI):

| Field | Type | Writers |
|---|---|---|
| `token_name`, `token_symbol`, `token_decimals` | string / string / u8 | init only |
| `total_supply`, `max_supply` | u128 | mint/burn only |
| `balances` | map1 address → u128 | transfer paths |
| `allowance_amounts` | map2 (owner, spender) → u128 | approval family, transfer_from |
| `allowance_expiries` | map2 (owner, spender) → u64 | approval family |
| `minter`, `manager`, `freezer` | address | role ops |
| `frozen` | map1 address → bool | freeze ext only (absent otherwise) |
| `registered` | map1 address → u128 (bond) | registration ext only |

Normative layout property: an ordinary `transfer` writes exactly
`balances[from]` and `balances[to]` and reads no shared mutable cell,
so disjoint-party transfers have disjoint write-sets under
slot-granular concurrency control.

### 9. Extensions

Extensions compile **out**, not off: a disabled extension contributes
no code, no storage reads, and no ABI entries. Enabled extensions are
enumerable from the artifact (`extension_flags`, plus the ABI shape).
Capabilities can never be added after deploy.

- **freeze** — per-account; `transfer`/`transfer_call`/`transfer_from`
  read `frozen[from]` and `frozen[to]` and revert `token:frozen`.
- **pause** — global brake; mutating token operations revert
  `token:paused` while set; distinct `pauser` role.
- **registration = "required"** — transfers to unregistered accounts
  revert `token:not_registered`. `register()` is `PAYABLE`, escrows
  an exact bond (excess reverts before any state write — and because
  the function is otherwise idempotent and validated up-front, no
  execution path both accepts value and reverts), records it in
  `registered[caller]`; `deregister()` pays the bond back via the
  native transfer host fn and zeroes the slot.
- **metadata_uri** — a mutable string custodied by `manager`;
  renouncing `manager` freezes it permanently.

### 10. Receiver protocol

A contract that accepts `transfer_call` deposits exports:

```
fn on_token_received(operator: Address, from: Address,
                     amount: u128, data: Vec<u8>) -> u32
```

Requirements: (a) the function MUST authenticate `caller()` against
an author-maintained set of accepted token contracts before acting —
the entry is publicly callable and unauthenticated calls MUST NOT be
treated as deposits; (b) it MUST return `ACK_TOKEN` to accept; any
other return or a revert rejects the transfer; (c) it MUST NOT carry
the `REENTRANT` attribute. Toolchain support (a manifest `[receiver]`
declaration generating the authenticated wrapper) is a Reference
Implementation deliverable; hand-written receivers meeting (a)–(c)
are conformant.

### 11. Events

Signatures (topic0 = Blake3 of the canonical signature string):

| Event | Signature | Indexed | Data (Borsh) |
|---|---|---|---|
| Transfer | `Transfer(address,address,uint128)` | from, to | `{amount: u128}` |
| Approval | `Approval(address,address,uint128,uint64)` | owner, spender | `{remaining: u128, expiry_wave: u64}` |
| RoleTransfer | `RoleTransfer(bytes32,address,address)` | role, new | `{previous: Address}` — role is the 32-byte `ROLE_*` identifier |
| Freeze | `Freeze(address,bool)` | account | `{frozen: bool}` |
| Registration | `Registration(address,bool,uint128)` | account | `{registered: bool, bond: u128}` |

The `Transfer` topic layout uses three of four topics; the fourth is
**reserved** for a future additive extension of this standard and
MUST NOT be used otherwise. Mint and burn are expressed as `Transfer`
with the zero-address sentinel; no separate Mint/Burn events exist.

### 12. Non-fungible surface (pts-n/1)

Storage: `owners` (map1 u64 → address), `balances` (map1 address →
u64), `token_approval` (map1 u64 → address; cleared on transfer),
`operators` (map2 (owner, operator) → bool), `token_uri` (map1 u64 →
string), `next_id` (u64), plus the shared metadata/role scalars.
`decimals` does not exist.

```
fn standard() -> String        // "pts-n/1"
fn owner_of(id: u64) -> Address
fn balance_of(owner: Address) -> u64
fn token_uri(id: u64) -> String
fn transfer_from(from: Address, to: Address, id: u64)
fn transfer_call(to: Address, id: u64, data: Vec<u8>) -> u32
fn approve(spender: Address, id: u64)
fn set_approval_for_all(operator: Address, approved: bool)
fn is_approved_for_all(owner: Address, operator: Address) -> bool
fn mint(to: Address, uri: String) -> u64    // minter role; returns the id
fn burn(id: u64)
```

`transfer_call` follows the identical settle-then-notify +
acknowledgement rules with `on_nft_received(operator, from, id,
data) -> u32` returning `ACK_NFT`. Events: `Transfer(from, to,
id)` with all three indexed; `Approval(owner, spender, id)`;
`ApprovalForAll(owner, operator, approved)`. Royalties are
out-of-scope (unenforceable by interface); no fused
fungible/non-fungible variant exists or will.

### 13. Conformance

An artifact conforms to `pts-f/1` iff (a) its `pyde.abi` section
contains exactly the generated surface for its declared extension set
— functions, attribute bits, parameter types, returns, events, and
state schema — and (b) it passes the byte-level conformance vectors
published with this PIP (canonical calldata → canonical state writes,
events, and reverts), including the malicious-receiver battery.
`otigen verify --standard pts-f <address|bundle>` performs both
checks. Because tokens are config-only, rebuild-and-byte-compare of
the manifest is the strongest verification and MUST reproduce for a
conformant generator version.

### 14. Activation

No consensus change, no fork, no engine modification. Activation is
a toolchain release: otigen ships generation + verification, the
conformance vectors freeze, and this PIP moves to Accepted when the
reference implementation merges.

## Rationale

- **Manifest type over a native module**: a native asset module
  reaches the same one-implementation property but costs engine
  surface Pyde does not need to take on; the toolchain already owns
  codegen across four languages, and artifact-embedded ABIs make
  conformance a static check. The chain stays token-ignorant.
- **Config-only over bundled custom logic**: allowing author code
  inside token modules would degrade reproducible verification to
  "conformant subset plus unreviewed extras" and reopen the
  drain-function-inside-a-conformant-token hole. Custom behaviour is
  a companion contract using the public surface; needs that genuinely
  belong inside the token enter as declared extensions in a future
  `pts-f/2`.
- **Expiring delta-allowances over `permit`-style signatures**: every
  scheme that relocated granting into off-chain signatures was
  phished at scale within months of deployment. Bounding lifetime
  and scope on-chain breaks drain economics structurally; platform
  sponsorship and commit-reveal already cover gasless and private
  flows without a new signature surface.
- **Settle-then-notify with a required acknowledgement over hooks or
  bare transfers**: pre-settlement hooks are the costliest exploit
  mechanism in token history; bare transfers strand funds. Notifying
  after full settlement, requiring `ACK_TOKEN` (which a fallback
  cannot accidentally produce), and keeping the plain path code-free
  captures the benefit of both without either failure mode.
- **`standard` on `[contract]` over per-kind manifest types**: an NFT
  is a token; `type` stays a stable two-value vocabulary and the
  standard family grows by value, keeping fused hybrids
  unrepresentable.
- **decimals default 9**: parity with native quanta (1 PYDE = 10⁹)
  gives the ecosystem one arithmetic mental model and makes exact
  wrapped-PYDE parity trivial.
- **`u32` acknowledgements, `bytes32` role ids, and a 3-field
  `Transfer`**: verified against the toolchain as it exists — the
  manifest type vocabulary has no 4-byte token (`bytes4` is
  deliberately rejected), so the acknowledgement is a `u32` whose
  little-endian bytes are the Blake3-derived tag (identical wire
  bytes); indexed variable-width event fields have no chain-committed
  hashing convention, so `RoleTransfer` carries a precomputed 32-byte
  role identifier; and an event's signature must list every declared
  field, so `Transfer` keeps exactly three fields to preserve
  byte-identical `topic0` with pre-PTS deployments — operator
  attribution lives in receipts, and richer transfer telemetry is
  additive `pts-f/2` territory (the reserved fourth topic).

## Backwards compatibility

No deployed-chain compatibility surface changes: PTS artifacts are
ordinary contracts. The `Transfer` event signature is byte-identical
to the existing example token, so subscriptions and explorer decoding
survive. Migration from the current `erc20-token` example: mutating
functions drop `-> bool` (revert-only semantics); revert strings move
to the canonical `token:*` codes; `approve(spender, amount)` remains
callable with bounded-TTL semantics; the `allowances` field name and
key shape are unchanged (the value widens to the Borsh
`{amount, expiry_wave}` pair, so slot-resolution tooling is
unaffected). The examples rename (`erc20-token` → `fungible-token`,
`erc721-token` → `nft-token`) with no wire impact.

## Security considerations

- **Codegen monoculture**: one generated implementation concentrates
  blast radius — a generator bug ships to every standard token and
  deployed artifacts are immutable. Countermeasures are normative
  deliverables: an independent audit of the generator, byte-level
  conformance vectors frozen before Accepted, and the
  malicious-receiver battery (including cross-function re-entry
  cases under the engine's per-function guard).
- **Receiver authentication**: `on_token_received` is publicly
  callable; the caller-authentication requirement (§10a) is normative
  because its omission is the documented drain class on comparable
  designs. The toolchain wrapper makes the check unwritable rather
  than merely documented.
- **Griefing via `transfer_call`**: a malicious receiver can revert
  (aborting atomically — the consent feature) or burn reserved gas
  (bounded by the caller's reservation and priced under no-refund
  economics; `MAX_CALL_DATA` bounds payload-driven amplification).
- **Value forfeiture**: with `register()` specified revert-free and
  every other function non-payable, no generated path can both accept
  native value and revert.
- **Proxy hole**: a delegate-call proxy in front of a conformant
  implementation can swap semantics later. Static conformance cannot
  close this; wallets MUST surface proxied tokens as prominently as
  freeze/mint capabilities.
- **CAS under commit-reveal**: `set_allowance_exact` can fail at
  reveal time; integrators must treat `token:allowance_changed` as a
  retriable condition, not corruption.

## Reference implementation

TBD during Draft. Planned in
[`pyde-net/otigen`](https://github.com/pyde-net/otigen): (1)
reference examples `fungible-token` and `nft-token` implementing the
surfaces on the current toolchain, with the AMM and marketplace
examples as reference receivers; (2) `type = "token"` generation from
one canonical implementation plus `otigen verify --standard`; (3) the
conformance vector suite and malicious-receiver battery.

## Copyright

Public domain via CC0.
