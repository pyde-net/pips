---
pip: 1
title: PIP Purpose and Process
author: zarah (@zarah-s)
type: Meta
status: Accepted
created: 2026-04-19
---

## Abstract

This PIP ratifies the Pyde Improvement Proposal (PIP) system itself. It codifies what a PIP is, how proposals move through states, and how protocol changes reach validators.

## Motivation

Pyde launches with a minimal on-chain governance surface (`MultisigTx`, `EmergencyPause`, signer rotation). All other protocol changes — new transaction types, consensus rule modifications, gas schedule updates, crypto primitive swaps — happen through social coordination rather than on-chain voting. A standardized process makes this coordination tractable.

The PIP system is borrowed in spirit from Ethereum's EIPs, Bitcoin's BIPs, and Rust's RFCs. These processes have successfully coordinated large-scale protocol evolution without on-chain voting, and a substantial share of Pyde contributors come from those ecosystems.

## Specification

### PIP format

Every PIP is a single markdown file in the `pips` repository with the filename `pip-NNNN-short-title.md`, where NNNN is zero-padded to four digits. The file begins with a frontmatter block specifying metadata. Required sections are defined in the repository's `README.md`.

### State machine

A PIP exists in exactly one of these states:

- `Draft` — PR merged, active iteration
- `Review` — substantive changes paused, final technical feedback
- `Last Call` — 14-day final comment period
- `Accepted` — terminal, reference implementation merged, awaiting activation slot
- `Rejected` — terminal, consensus against adoption
- `Withdrawn` — terminal, author retracted

Transitions are tracked via PRs that update the PIP's `status` frontmatter field. Every state transition MUST be accompanied by a GitHub Release announcement on this repository so validators and community members have a fair chance to respond.

### PIP number assignment

PIP authors submit proposals with `pip: XXXX` as a placeholder in frontmatter. A PIP editor assigns the next available number at PR merge time. This prevents two simultaneous submissions from claiming the same number.

### Rough consensus

PIP acceptance is determined by rough consensus as judged by PIP editors. Rough consensus means most participants in the discussion are satisfied and no substantive technical objection remains unaddressed. It is not a vote count. A single unanswered concern from any participant can block advancement if it identifies a genuine technical issue.

**Acceptance does not guarantee implementation.** Client teams independently decide what to ship. An `Accepted` PIP that no client team implements is effectively moot — if an implementation doesn't reach validators, there's nothing to activate at the specified slot. The PIP process produces a binding specification for what *would* ship if shipped; it does not compel any team to ship it.

### PIP editors

PIP editors are a small rotating group responsible for procedural coordination: assigning PIP numbers on PR merge, checking that required sections are present, assessing whether discussion has converged enough to advance the PIP, and publishing GitHub Release announcements at state transitions. Editors do not have unilateral authority to accept or reject a PIP.

Initial editors are listed in the repository `README.md` and updated via Meta-type PIPs.

### Primary discussion venue

The primary discussion venue for any PIP is the pull request for that PIP on this repository. Substantive comments and decisions must appear there so there's a single auditable record. Companion issues are allowed for long-running parallel topics but conclusions must be mirrored back to the PR. Real-time channels (Discord, mailing lists) are allowed for synchronous discussion, but any decision reached there must be documented in the PR before affecting state.

### Protocol activation

`Standards Track` PIPs that affect consensus require a hard fork. The accepted PIP MUST specify an activation slot at least 6,500,000 slots (≥ ~30 days at the wave-commit cadence) after the `Accepted` transition. The main client ships an implementation behind the activation-slot gate and is released before the activation slot so validators have time to upgrade.

Non-upgraded validators fork themselves off at the activation slot. Their stake remains but produces no rewards on the dead fork.

### Reference implementation timing

A PIP's reference implementation MUST be merged into the main `pyde` repository (possibly behind a feature flag) before the PIP advances from `Last Call` to `Accepted`. This prevents accepting specifications that turn out to be impractical to implement.

### Relationship to on-chain governance

On-chain governance primitives (`MultisigTx`, `RotateMultisig`, `EmergencyPause`, `EmergencyResume`) do not modify the protocol. They only operate within it: moving treasury funds, freezing block production during emergencies, and rotating the multisig signer set. Protocol rule changes always go through the PIP + hard fork path.

## Rationale

On-chain governance has repeatedly proven fragile during the early life of a chain. Low voter turnout enables cheap capture; formalized voting mechanisms make disputes legible and therefore easier to litigate; failed governance proposals are nearly impossible to undo without a hard fork anyway.

Social consensus via PIP + voluntary client upgrade is imperfect but has shipped reliably across Bitcoin, Ethereum, and Rust for over a decade combined. The PIP system also leaves open the option of adding on-chain governance later via a future PIP, while the reverse (removing deployed on-chain governance) is significantly harder.

The "acceptance does not guarantee implementation" clause is deliberate. It recognizes that no document can force code to ship. Rather than pretend otherwise, the process is honest about where the real gating step lives — with client teams deciding what goes into binaries. This mirrors how Ethereum actually works despite EIP acceptance looking more deterministic on paper.

## Backwards compatibility

This is the first PIP. There is nothing to be backwards compatible with.

## Security considerations

The PIP system itself is a social process hosted on GitHub. Risks:

- **GitHub censorship**: proposals or discussion could be restricted by GitHub. Mitigation: the repository content can be mirrored to IPFS or self-hosted git if needed; the process doesn't depend on GitHub-specific features beyond PRs and Releases.
- **Editor capture**: a PIP editor could refuse to advance a proposal or publish a Release announcement. Mitigation: editors rotate; their authority is procedural; participants can open a Meta-type PIP to change the editor set.
- **Review brigading**: an adversary could flood a PIP discussion to fabricate consensus. Mitigation: rough consensus is about unresolved technical objections, not vote count; editors judge based on substance.
- **Activation-window rug-pull**: an adversary could try to push an accepted PIP to activation faster than validators can upgrade. Mitigation: the 6,500,000-slot (≈ 30 day) minimum window is required by this PIP, not a guideline.

None of these are protocol-level risks. Governance failures show up as coordination failures (e.g., failed forks), not consensus failures.

## Reference implementation

This PIP is self-implementing — the `pips` repository and its `README.md` constitute the implementation.

## Copyright

This PIP is licensed under [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
