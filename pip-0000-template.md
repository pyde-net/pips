---
pip: XXXX
title: <concise title, ~50 chars>
author: <your name, email, github handle>
type: <Standards Track | Meta | Informational>
status: Draft
created: <YYYY-MM-DD>
requires: <optional, PIP numbers this depends on>
---

## Abstract

One paragraph. A reader should understand what changes without reading further.

## Motivation

Why should Pyde adopt this? What problem does it solve? What's broken today that this fixes?

## Specification

The technical meat. Precise enough that a second implementer could build it from this section alone. Include as applicable:

- New/modified transaction types, fields, encodings
- New/modified state keys + discriminators
- New/modified consensus rules
- Gas cost changes (with benchmark data)
- Activation criteria (slot number, version check, etc.)

Code snippets where helpful. Reference the existing crate structure in `pyde/crates/`.

For Standards Track PIPs, the activation slot MUST be at least 6,500,000 slots (≥ ~30 days at the wave-commit cadence) after the `Accepted` transition to give validators time to upgrade.

## Rationale

Why this design? What alternatives were considered? Why were they rejected?

## Backwards compatibility

- Does this break existing clients? How?
- What's the migration path for existing users/contracts?
- If it's a hard fork, what's the activation mechanism?

## Security considerations

- New attack surface introduced?
- Mitigations in the design?
- Risks during/after activation?
- Post-quantum properties preserved?

## Reference implementation

Link to a PR in `zarah-s/pyde` that implements this PIP. May be TBD during `Draft` state, but must be merged into the main `pyde` branch (possibly behind a feature flag) before this PIP advances to `Accepted`.

## Copyright

This PIP is licensed under [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
