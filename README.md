# PHASE — reversible phase-state computing

**A computational paradigm for ordinary machines.** Computation is treated as
motion of *phase state*: inside a phase every operation is reversible (time
runs both ways, rollback is free), while erasure, input and output happen only
at phase boundaries and are paid for there. Determinism is sealed by a
hash-chain; state travels in `.eml` capsules; **failure is a control-flow
branch — fail = un-run.**

This is not a new programming language or another framework. It is a
**discipline** — a way to structure computation, memory, communication and
failure — that runs on the x86 hardware you already have and cures (with
software) a large class of von Neumann's ailments: snapshot costs, WAL logs,
GC pauses, non-determinism, crash-then-restart recovery, state migration
pain.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![impl](https://img.shields.io/badge/impl-phase--vm-blueviolet.svg)](https://github.com/kostyk348/phase-vm)
[![tests](https://img.shields.io/badge/tests-39%20passing-brightgreen.svg)]()

> Docs: [ARCHITECTURE.md](ARCHITECTURE.md) (full spec) ·
> [X86.md](X86.md) (von-Neumann ailments & PHASE's own, with workarounds) ·
> [TYPICAL.md](TYPICAL.md) (paradigm for everyday tasks) ·
> Reference implementation: **[phase-vm](https://github.com/kostyk348/phase-vm)**

---

## 1. The problem with the machine we have

In the classical von Neumann machine:

- **Time is destructive.** `a = b + c` erases the old `a`. To go back you need
  external snapshots, write-ahead logs, or copy-on-write filesystems — all
  proportional to the *bytes* you touched.
- **Failure means restart or restore.** State is half-broken; recovery is a
  big hammer.
- **Execution is non-deterministic to observe.** Same input does not guarantee
  a reproducible trajectory, which makes debugging, testing and networking
  (reconciliation) expensive.
- **Memory is passive, GC and malloc are cold.** Lifetime management is
  per-object, scattered, and pauses.
- **State does not travel.** Migrating a live process or sealing a
  reproducible artifact requires bespoke machinery.

Some of these need new silicon (Landauer energy, the memory wall). Most of
them are *discipline problems* — and discipline is software.

## 2. The idea in one picture

```
open a phase                    mark / fresh frame / arena region
        │
        ▼
inputs at the boundary          data, disturbances, network events — irreversible, once
        │
        ▼
reversible kernel  F            every instruction: one destination, sources stay intact
        │                          → the inverse is computable from the current state
        │                          → zero logs, zero snapshots, inside the phase
        ▼
validator at the boundary       invariants checked AFTER applying (cascades become visible)
        │
        ├── ok    ──► commit: seal a hash-chain step → output / capsule
        └── fail  ──► rollback: F⁻¹ over O(steps) → try another candidate
```

**Why "sources stay intact" is the whole trick.** A destructive machine must
log what it erased. PHASE never erases inside a phase: `r3 += r0` keeps `r0`,
so to undo it you only need the *current* values — `r3 -= r0`. The source *is*
the log, and it costs nothing. Erasure (`set`, `mset`, I/O, external input) is
therefore confined to boundaries and is metered there, once per phase.

## 3. What it gives you (the three honest axes)

| axis | mechanism | measured consequence |
|---|---|---|
| **A · Reversibility** | bijective ops inside a phase | rollback costs **O(steps taken), not O(bytes of state)** |
| **B · Meet-in-the-middle** | bidirectional search where structure allows | sqrt-style reduction (demo: 1.6× fewer BFS expansions) |
| **C · Boundary discipline** | erasure/IO/commit only at phase edges | arenas instead of GC, WAL-less transactions, capsule migration |

These are **independent** wins. Reversibility is not a search speedup; MITM
is not a memory trick. The paradigm combines them under one rule: *everything
is a phase.*

## 4. The layered architecture

| L | layer | invariant | status |
|---|---|---|---|
| **L5** | Applications on phases | tick / agent-step / capsule = phase | partial — PC1, netcode, A/B, parser |
| **L4** | Capsules `.eml`/`.vcf` | self-contained, verifiable without trust | core done (`src/cap.rs`) |
| **L3** | Semantics & control | journal = decisions only, never state | done (`src/ctl.rs`) |
| **L2** | Phase memory (arenas) | memory = peak of phase, not sum | done (`src/alloc.rs`) |
| **L1** | Phase machine | `F⁻¹(F(S)) == S` inside a phase | done (phase-vm) |
| **L0** | Bijective core | one destination, sources intact | done (phase-vm ISA) |

Readable deep spec in [ARCHITECTURE.md](ARCHITECTURE.md).

## 5. Measured (release, normal x86, this machine)

| claim | number |
|---|---|
| rollback is O(steps), not O(bytes) | reverse ~**6.8 ns/leaf, flat** for state 64 B → 1 MB; snapshot 7.8 ns → **25 µs @ 1 MB** |
| phase arena rollback | O(1) `truncate`; alloc ~1.1 ns/op flat at any K vs malloc+free 4.8 → 35 ns/op → **4–20×** on phase-shaped loads; **zero per-object headers** |
| decision journal | 500k-iteration data-dependent `while` → **1 entry = 8 B** (naive step trace: 8 MB) |
| WAL-less transaction batch | 35.8 % batches aborted (cascade overdrafts), **0 copies** vs 39 MiB snapshots |
| speculative tick control | 300k ticks, deterministic; MPC-style snapshots 32 MiB → **0**; zero-trace asserted |
| ensemble (worst-case) | 6 plant variants in one state; `|v| ≤ VMAX` held on every lane every tick |
| rollback netcode | 142k reconciliations; input log 240 KB vs world snapshots 3.84 MB (**16×**); hash-chain of all 60 001 tick boundaries equals ground truth |
| AOT compilation | 0.06 vs 6.08 ns/leaf forward (**94.8×**), reverse 94.9× |
| capsule migration | 612 B `.eml`; receive-side verify without trust: state-hash → reverse → boundary-hash → re-forward |
| reversible synthesis | shortest kernel from I/O samples; xor-swap found, random target rediscovered (200 held-out) |
| typical tasks | A/B: 0 copies vs 32 MB clones over 500k rounds; parse: 54 515 O(1) rollbacks vs 327 090 individual frees |

Every number above is produced by a runnable example with asserts —
`cargo run --release --example <name>` in **phase-vm**.

## 6. Honest limits (we do not promise these)

- Reversibility lives on **bits and integers**. IEEE-754 is not invertible
  (NaN, rounding) — floats stay out of kernels, on the boundary.
- "Zero logs" holds **inside a phase**. Crossing a `set`/`mset` boundary needs
  a checkpoint — that is the price of erasure, paid once per phase.
- Data-dependent branching needs a **decision journal** — O(decisions), never
  O(state) (measured: 8 B for a 500k loop). The journal doubles as the
  capsule's "trajectory".
- A phase arena wins on **phase-shaped lifetimes** (tick/request/frame/
  transaction). Long-lived objects with random interleaved lifetimes → keep a
  normal allocator for those (two-tier memory).
- This is a software discipline on von Neumann silicon. We do **not** cure the
  energy cost of erasure (that needs adiabatic hardware) or the memory wall.
- PC1 is a paradigm demonstrator, **not** an audited cipher.
- "Non-Hermitian topology", "skin effect" etc. from the manifesto are
  **metaphors** for directional scheduling — real physics stays in photonics.

## 7. Typical tasks in the paradigm

What ordinary programming looks like under this discipline — mapping for ETL
batches, A/B selection, parsing with errors, retries/sagas, data migration,
untrusted input, game/server ticks, sort-with-undo, encryption-at-rest:
see **[TYPICAL.md](TYPICAL.md)**. The pattern is always:

> allocate in a phase → transform reversibly → **validate at the boundary** →
> commit or roll back.

## 8. The implementation: phase-vm

[github.com/kostyk348/phase-vm](https://github.com/kostyk348/phase-vm) — a
reversible register machine in Rust, zero dependencies, 39 tests.

```
check  classify leaves (invertible / boundary / aliasing)
run    forward · runrev backward from a state
rev    forward then unwind (--steps K) · dbg interactive time-travel
roundtrip  property fuzz F⁻¹(F(S))==S
bench  ns/leaf · cipher PC1 (decrypt = reverse)
```

Modules: `inst` bijective ISA · `machine` streaming exec/reverse · `alloc`
phase arena · `ctl` decision-journal control flow · `cap` `.eml` capsule ·
`aot` C codegen → native (~95×) · `synth` reversible kernel synthesis.
Run all demos:

```bash
git clone https://github.com/kostyk348/phase-vm && cd phase-vm
cargo run --release --example alloc_bench     # arena vs malloc
cargo run --release --example batch_tx        # WAL-less transactions
cargo run --release --example spec_control    # speculative tick control
cargo run --release --example ensemble_control# worst-case ensemble
cargo run --release --example rollback_netcode# reconciliation, no snapshots
cargo run --release --example aot_bench       # native compilation (~95×)
cargo run --release --example capsule         # .eml migration, verify w/o trust
cargo run --release --example ctl_demo        # decision journal (8 B / 500k loop)
cargo run --release --example mitm_bfs        # meet-in-the-middle search
cargo run --release --example synth_demo      # reversible kernel synthesis
cargo run --release --example typical_ab      # A/B without state copies
cargo run --release --example typical_parse   # parsing with O(1) recovery
```

## 9. Where these ideas came from

PHASE is assembled from the reversible-computing literature (Landauer 1961,
Bennett 1973, Fredkin–Toffoli 1982, Margolus 1984) and from **fifteen years of
the author's own systems** — SINT (semantic registers, hash-chains, quorum),
DSO (deterministic ticks), MPTC (tick-aligned arenas), DSA (data-oriented
audio), DSO Control (ensemble stability), mime-os (`.eml` cells), DataTrace,
KCC. See the puzzle table in [ARCHITECTURE.md](ARCHITECTURE.md): every past
project is a piece of the whole — substrate or direction.

## 10. Roadmap

- [x] L0 bijective core · L1 phase machine · L2 arena · L3 decision journal
- [x] L4 capsule core · L5 PC1, netcode, typical-task examples
- [x] P2 AOT (~95×) · P10 wide-lane measurement · axis B MITM demo · synthesis
- [ ] a real end-to-end project written entirely in the paradigm
- [ ] SoA wide kernels (AVX2 in depth), `.vcf` manifests to full actor model
- [ ] agent-step rollback for the SINT cognitive stack (dogfooding)

## License

MIT — see [LICENSE](LICENSE).
