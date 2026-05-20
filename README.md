# Pyde Improvement Proposals (PIPs)

PIPs are the primary mechanism for proposing changes to the Pyde protocol and surrounding governance processes. This repository hosts the text of each proposal and the discussion around it.

## When to write a PIP

- **Yes**: changes to consensus rules, transaction types, gas costs, state layout, crypto primitives, or governance procedures.
- **Yes**: new standards that multiple clients/tools should agree on (e.g., wallet address formats, RPC methods).
- **No**: bug fixes in a specific client — those are regular pull requests in the main repo.
- **No**: one-off treasury spends — those go on-chain directly via `MultisigTx`, linked from an accepted PIP.

## PIP types

| Type | Purpose |
|---|---|
| **Standards Track** | Changes that affect the protocol. Activation requires a hard fork. |
| **Meta** | Changes to the PIP process itself (this repo, how proposals are reviewed, etc.). |
| **Informational** | Design notes, guidelines, or context — no implementation mandate. |

## Lifecycle

```
Draft  ──▶  Review  ──▶  Last Call (14 days)  ──▶  Accepted  ──▶  [implemented + activated]
  │           │                                        │
  └───────────┴─── Withdrawn ─────────────────────────┘
                                                       │
                                                  Rejected (during Review or Last Call)
```

- **Draft**: initial PR. Author iterates on feedback.
- **Review**: PR merged; community + core team actively discussing.
- **Last Call**: 14-day final comment period. No substantive changes during this window.
- **Accepted**: final. Reference implementation merged into the main `pyde` repository (possibly behind a feature flag) before this transition.
- **Rejected / Withdrawn**: terminal states. Withdrawn = author pulled it. Rejected = community consensus against.

## How to propose

1. Fork this repo.
2. Copy `pip-0000-template.md` to `pip-XXXX-short-title.md`. Leave `pip: XXXX` as a placeholder — the PIP editor assigns the final number when merging your PR, so two simultaneous submissions don't collide.
3. Fill in every required section. Incomplete proposals are closed without prejudice; reopen once sections are filled.
4. Open a PR. The PR description should include a 2-3 sentence summary.
5. Address feedback in the PR itself — this is the primary discussion venue (see below). For long-running parallel topics, open a companion issue and link it from the PR.
6. Once the PR is merged into `main`, your PIP is officially in `Draft` state.

## Required sections (every PIP must have all)

- **Abstract** — one paragraph summary
- **Motivation** — why this change is worth making
- **Specification** — precise technical description
- **Rationale** — why this design over alternatives
- **Backwards compatibility** — breaking changes + migration path
- **Security considerations** — attack surface, mitigations
- **Reference implementation** — link to code in the main `pyde` repo (can be TBD during Draft; must be merged before Accepted)

## Discussion venues

- **Primary**: the PR for each PIP. Substantive comments + decisions must appear there so there's a single auditable record.
- **Long-running topics**: a companion issue on this repo, linked from the PIP's PR. Issues exist to keep PR threads readable; conclusions are mirrored back to the PR.
- **Real-time channels** (Discord, etc.): allowed for synchronous discussion, but any decision reached there must be documented in the PIP's PR before affecting state.

## How decisions are made

Pyde PIPs aim for **rough consensus**: most participants in the discussion are satisfied and no substantive technical objection is unanswered. It is not a vote count. A single unanswered concern from any participant can block advancement if it identifies a genuine technical issue.

In practice:
- A PIP editor (rotating role) assesses whether discussion has converged.
- The editor proposes advancing the PIP to the next state (Review → Last Call → Accepted).
- If any participant raises a substantive technical objection, advancement stalls until it's addressed.
- Core team has no unilateral merge authority for protocol changes — they participate as peers.

**Acceptance does not guarantee implementation.** Client teams independently decide what to ship. An Accepted PIP that no client team implements is effectively moot — if an implementation doesn't reach validators, there's nothing to activate. The PIP process produces a binding specification for what *would* ship if shipped; it does not compel any team to ship it.

## How Accepted PIPs reach validators

1. A client team implements the PIP in the main `pyde` repository, merged before `Accepted` status.
2. The accepted PIP specifies an **activation slot**. For Standards Track PIPs, the activation slot must be at least **6,500,000 slots** (≥ ~30 days at the ~500ms median commit cadence) after the PIP enters `Accepted`. This gives validators time to upgrade binaries.
3. A client release ships containing the implementation behind the activation-slot gate.
4. Validators voluntarily upgrade.
5. At the activation slot, upgraded nodes start enforcing the new rules. Non-upgraded nodes fork themselves off — their stake remains but produces no rewards on the dead fork.

## Announcement channels

When a PIP transitions state (especially into `Last Call` or `Accepted`), the editor publishes a notice so validators and the community have a fair chance to object or prepare. Official channels:

- **This repository's GitHub Releases page** — primary source of truth, one release per state transition.
- **TBD at mainnet launch**: a moderated announcement mailing list and/or social channel. Once established, this section will be updated via a Meta-PIP.

Any state transition without a corresponding Release announcement is considered invalid — the editor must publish notice before a PIP can move forward.

## Relationship to on-chain governance

On-chain governance (treasury spends, emergency pauses, signer rotations) uses the `MultisigTx`, `RotateMultisig`, `EmergencyPause`, and `EmergencyResume` transaction types. These are the *only* on-chain governance primitives.

Protocol changes (new tx types, consensus rules, gas schedules) happen via PIP + hard fork — **not** via on-chain vote. The multisig cannot modify the protocol, only operate within it.

A treasury spend's on-chain `MultisigTx` should reference the PIP that rationalized it in `tx.data_digest = hash(pip_file_contents)` so the spend is auditably linked to its proposal.

## PIP editors

PIP editors coordinate the process: assign numbers on PR merge, check that required sections are present, assess whether discussion has converged enough to advance the PIP, and publish Release announcements at state transitions. Current editors:

- TBD — to be populated at mainnet launch via a Meta-PIP.

Editors have no unilateral authority over acceptance. Their role is procedural.

## License

PIP text and this repository are licensed under [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
