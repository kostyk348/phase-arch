# PHASE — reversible phase-state computing architecture

Blueprint for a von-Neumann-challenging paradigm assembled from the manifesto
ideas (reversible phase computation · meet-in-the-middle · boundary
discipline · `.eml/.vcf` capsules) and 15+ past systems (SINT, DSO, MPTC,
DSA, DSO Control, mime-os, DataTrace, KCC, …).

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![doc](https://img.shields.io/badge/architecture-ARCHITECTURE.md-green.svg)](ARCHITECTURE.md)
[![impl](https://img.shields.io/badge/impl-phase--vm-blueviolet.svg)](https://github.com/kostyk348/phase-vm)

## Core thesis

> Computation is motion of phase state. **Inside a phase all operators are
> bijective** — time is traversable both ways, rollback is free. Erasure,
> input and output exist **only at phase boundaries** and are paid there
> explicitly. A validator decides commit/rollback at the boundary. Determinism
> is sealed by a hash-chain. State travels in capsules (`.eml/.vcf`).
>
> **Failure is a control-flow branch: fail = un-run.**

## Layers (invariants, not modules)

| L | name | invariant | status |
|---|---|---|---|
| L5 | Applications on phases | tick/agent-step/capsule = phase | partial (PC1, netcode next) |
| L4 | Capsules `.eml/.vcf` | self-contained, verifiable anywhere | design (substrate: mime-os) |
| L3 | Semantics & control | journal = decisions only, never state | **building (decision journal)** |
| L2 | Phase memory (arenas) | peak of phase, not sum | done (`src/alloc.rs`) |
| L1 | Phase machine | `F⁻¹∘F = Id` inside a phase | done (phase-vm) |
| L0 | Bijective core | one destination, sources intact | done (phase-vm ISA) |

Three honest axes: **A reversibility** (free rollback O(steps)) ·
**B meet-in-the-middle** (sqrt-factor where structure allows) ·
**C boundary discipline** (memory/communication/failure).

Full spec: **[ARCHITECTURE.md](ARCHITECTURE.md)** — axioms (what is measured
and what is honestly off-limits), layer invariants, the 20-project map,
life-cycle of a phase, past-systems puzzle, build order, limitations.

## Reference implementation

[**phase-vm**](https://github.com/kostyk348/phase-vm) — reversible register
machine: `F⁻¹(F(S))==S` bit-exact, reverse ~6.8 ns/leaf at any state size,
arena rollback O(1), PC1 cipher where decrypt = backward run.

## License

MIT — see [LICENSE](LICENSE).
