# Confidential sBPF on Logos

> Run **confidential Solana programs** — Solana bytecode (sBPF) interpreted inside Logos's
> privacy-preserving zkVM, operating on **shielded inputs that never become public**, with the
> verified result settled back on Solana.
>
> The only way to run Solana logic over data that can never be made public.

---

## Table of Contents

- [Core Insight](#core-insight)
- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Why Each Piece Is Necessary](#why-each-piece-is-necessary)
- [Scope Discipline](#scope-discipline)
- [Competitive Positioning](#competitive-positioning)
- [Cost Model](#cost-model)
- [Ecosystem Benefits](#ecosystem-benefits)
- [Risks](#risks)
- [Decision Experiments](#decision-experiments)
- [Staged Build Plan](#staged-build-plan)
- [Resources](#resources)
- [Pitch](#pitch)

---

## Core Insight

On normal Solana, programs are verified by **re-execution** — validators simply run them again.
This only works because all inputs are public.

**You cannot re-execute a program whose inputs are private.** If inputs are secret, no validator
can re-run it to check the result. The *only* way to trust a private computation is a **proof**, and
the *only* way to keep the inputs private end-to-end is a **privacy-preserving zkVM**.

That is the gap. Logos has a privacy layer. Therefore Logos is **not optional — it is the mandatory
ingredient**. Remove Logos and the value proposition collapses entirely.

---

## How It Works

### Actors

| Actor | Role |
|---|---|
| **Developer** | Writes a Solana program, marks certain inputs as confidential |
| **User** | Owns private data (balance, bid, identity attribute) in a Logos shielded account |
| **Your platform** | The middleware: SDK + coordinator program + Logos guest |
| **Solana** | Settlement layer where the developer's program and final verification live |
| **Logos (LEZ)** | The confidential execution layer |

### Lifecycle

1. **Developer writes a confidential Solana program.** They use familiar Solana tooling
   (Anchor/Rust → sBPF) but annotate which inputs are confidential via your SDK. Example logic:
   "return true if the user's private balance ≥ threshold." Compiles to normal sBPF bytecode.

2. **User's private data lives shielded on Logos.** The sensitive value sits in a Logos shielded
   account — encrypted, never on public state. (The LP-0013 shielded-account mechanism.)

3. **A request is initiated.** The user or dApp triggers a confidential computation: "run program X
   over my shielded value with public parameter threshold = T." Routed to your platform.

4. **Logos interprets the sBPF over private data.** Inside Logos's privacy-preserving zkVM:
   - The sBPF interpreter (the guest) loads the developer's bytecode
   - The private input (shielded balance) is fed as a private witness — never revealed
   - The public input (threshold) is supplied openly
   - The interpreter executes fetch-decode-execute over the secret data
   - Crypto syscalls route to accelerators (sha256, etc.) to keep cost down

5. **Proof generation + compression.**
   - RISC Zero records the execution trace → STARK proof
   - The proof is wrapped to Groth16 (small, Solana-verifiable)
   - The journal (public outputs) contains ONLY the result + public params — never the private input

6. **Return to Solana.** The Groth16 proof + journal + program image-ID are submitted to your
   coordinator program on Solana.

7. **On-chain verification + settlement.**
   - The coordinator verifies the Groth16 proof via Solana's `alt_bn128` syscalls (cheap)
   - It reads the verified result from the journal
   - It forwards the verified result to the developer's Solana program
   - The developer's program acts on a result it can trust, derived from data it never saw

8. **Solana finalizes the block** containing the settled result.

### The Data-Privacy Invariant

- **Private input** (balance): lives shielded on Logos → fed as private witness → never in any
  journal, never on Solana, never on public Logos state.
- **Public output** (result): in the journal → verified on Solana.
- **The proof attests:** "this exact bytecode, run over some private input satisfying the shielded
  commitment, produced this public result" — **without revealing the input.**

---

## Architecture

```
DEVELOPER
  writes Solana program (sBPF), marks inputs confidential via your SDK
        |
USER's private data --> shielded on Logos (encrypted, off public state)
        |
        v
+----------------------------------------------------------------+
| LOGOS (LEZ) -- THE IRREPLACEABLE LAYER                          |
|                                                                |
|  +------------------------------------------------+            |
|  | Your sBPF INTERPRETER (the guest)              |            |
|  |   - loads developer's bytecode                 |            |
|  |   - private witness: shielded balance (secret) |            |
|  |   - public input: threshold                    |            |
|  |   - fetch->decode->execute over secret data    |            |
|  |   - crypto syscalls --> ACCELERATORS           |            |
|  +------------------------------------------------+            |
|        |                                                       |
|        v                                                       |
|  STARK proof --> wrap to GROTH16                               |
|  journal = public result ONLY (no private input)              |
+-------------------------------+--------------------------------+
                               | groth16 proof + journal + image_id
                               v
+----------------------------------------------------------------+
| SOLANA -- SETTLEMENT                                            |
|  Your COORDINATOR program:                                     |
|    - verify_groth16(proof, journal, image_id)  [alt_bn128]     |
|    - read verified result from journal                         |
|    - forward to DEVELOPER's program                            |
|  Developer's program acts on trusted result                    |
+----------------------------------------------------------------+
```

---

## Why Each Piece Is Necessary

The "no part is decorative" test:

| Component | Why it's required | What breaks without it |
|---|---|---|
| **Logos privacy layer** | Keeps private inputs secret end-to-end | Inputs leak → just vanilla RISC Zero → no moat |
| **sBPF interpreter** | Lets devs use real Solana bytecode/tooling | Devs must rewrite in a DSL → smaller market |
| **Accelerators** | Keep crypto-heavy confidential logic affordable | Crypto syscalls explode the trace → too slow |
| **Groth16 wrap** | Makes the proof cheap enough for Solana | STARK too big → Solana can't verify |
| **Solana coordinator** | Anchors trust where the dev's program lives | Result has nowhere trustworthy to land |

---

## Scope Discipline

**Do NOT promise:** interpreting arbitrary, large, business-logic-heavy DeFi programs. Interpreter
overhead (50-100x per instruction) makes that impractical.

**DO target:** small **confidential** programs — the class that:

1. Is small enough that interpreter overhead is tolerable
2. *Cannot* run on public Solana (private inputs)
3. *Must* use a privacy zkVM (Logos)
4. Is often crypto-adjacent → accelerators help most exactly here

**Beachhead use cases:**

- Private eligibility / threshold checks (balance ≥ X, age ≥ 18)
- Sealed-bid auctions (bids stay secret until settlement)
- Private balance / solvency proofs
- Confidential KYC / identity predicates
- Private voting / allowlist membership

---

## Competitive Positioning

| | Arcium | This project |
|---|---|---|
| Trust model | MPC (multi-party computation) | ZK proofs |
| Programming model | Arcium DSL (`#[encrypted]`) | Real Solana sBPF + annotations |
| Privacy source | MPC network | Logos shielded zkVM |
| Verification | Attestation | Groth16 on Solana |

**Differentiation:** ZK (not MPC) confidential compute, over actual Solana bytecode, with Logos
privacy. A different trust model and more native to Solana tooling.

---

## Cost Model

Two sources of proving cost — understanding the split is essential:

1. **Interpreter overhead** — fetch/decode/dispatch per sBPF instruction. The dominant cost.
   **No accelerator fixes this.** Only scope discipline (small programs) and eventually JIT/AOT
   compilation address it.

2. **Leaf crypto** — sha256, signatures, etc. **Accelerators crush this.** Confidential programs
   are disproportionately crypto-heavy, so accelerators help more than average here.

**Key implication:** the proving cost is driven by *instructions executed*, not bytecode loaded.
Unexecuted bytecode just sits in the program memory region and costs almost nothing. Optimization
must target *executed work*, not binary size.

---

## Ecosystem Benefits

**Solana developers get:**

- Confidential computation while staying on Solana
- Use of familiar Solana tooling (not a foreign DSL)
- Results verified trustlessly on Solana

**Logos gets:**

- Becomes the confidential-compute layer for crypto's largest dev ecosystem
- Real usage / fees routed through its privacy zkVM
- A flagship use case that *requires* its privacy differentiator
- A bridge bringing Solana's developer base into the Logos ecosystem

---

## Risks

Go in clear-eyed:

1. **Interpreter overhead may be too high** even for small programs — must measure (Experiment 1).
2. **Shielded-value-as-private-witness may not be supported** by current LEZ tooling — must verify
   (Experiment 2).
3. **Groth16 export from LEZ** may not be wired up — must verify (Experiment 3).
4. **Solana ↔ LEZ RISC Zero version mismatch** could complicate the Groth16 verifier.
5. **Market demand unproven** — validate that Solana devs actually want confidential compute.
6. **Competing with funded teams** (Arcium) — niche must be defended on the ZK-vs-MPC +
   Solana-native axes.
7. **Solo founder, frontier tech, two ecosystems** — heavy. A working PoC de-risks enormously.

---

## Decision Experiments

Run these **before** committing. Three small experiments, in order. Each can confirm or kill the
project cheaply.

### Experiment 1 — Interpreter Overhead (the viability number)

- **Question:** How expensive is interpreting even trivial sBPF inside RISC Zero?
- **Test:** Write a minimal sBPF interpreter as a RISC Zero guest. Feed it the simplest sBPF program
  (add two numbers, no syscalls). Run under `RISC0_DEV_MODE=0` (or use the executor for a cycle
  count). Record cycles.
- **Decision:** Extrapolate — if "add two numbers" costs N cycles of interpreter overhead, estimate
  a 100-instruction confidential predicate. Is the proving time seconds (workable) or
  minutes-to-hours (impractical)?

### Experiment 2 — Shielded Value as Private Witness (the moat)

- **Question:** Can a Logos shielded-account value be fed as a private input to a guest, and proven
  over, without appearing in the journal or on-chain?
- **Test:** Take a shielded value (LP-0013 mechanism), feed it as a private witness to a guest that
  outputs only a boolean. Inspect the journal — confirm the value is absent.
- **Decision:** If the shielded value genuinely stays private end-to-end → the moat is real. If it
  leaks or isn't supported → the privacy claim fails and the design must change.

### Experiment 3 — Groth16 Verified on Solana (the settlement seam)

- **Question:** Can LEZ emit a Groth16 proof that Solana's verifier accepts?
- **Test:** Produce a Groth16-wrapped receipt from a LEZ guest (`ProverOpts::groth16()`), then verify
  it on Solana using RISC Zero's Solana verifier (`alt_bn128`).
- **Decision:** If Solana verifies it → the settlement chain closes. If version mismatch or
  unsupported → resolve before building.

**Gate:** All three pass → strongest-formed version of the vision, with Logos essential. Any fail →
you've learned the exact blocker cheaply, and you have a precise question for the Logos team.

---

## Staged Build Plan

| Stage | Goal |
|---|---|
| **Stage 0** | Experiments 1-3 (above) |
| **Stage 1** | Interpret one tiny sBPF predicate over one shielded input; measure + demo |
| **Stage 2** | Add crypto accelerators (sha256, signatures) |
| **Stage 3** | Expand sBPF opcode + minimal-syscall coverage for confidential programs |
| **Stage 4** | Pitch-worthy demo: a real confidential program (sealed-bid auction / private eligibility) over shielded Logos data, verified on Solana |
| **Stage 5** | SDK + developer annotations so external devs can build confidential Solana programs |

---

## Resources

- [`solana-labs/rbpf`](https://github.com/solana-labs/rbpf) — reference sBPF VM (interpreter design)
- [RISC Zero `risc0`](https://github.com/risc0/risc0) — guest/host model, `ProverOpts`, accelerators
- RISC Zero Solana verifier (Groth16 / `alt_bn128`) — settlement seam
- LP-0013 work (your own) — shielded accounts, `PrivacyPreservingTransaction`, LEZ guest patterns
- [Arcium docs](https://docs.arcium.com) — competitive reference for confidential-compute UX
- "Crafting Interpreters" (Bob Nystrom) — interpreter loop design

---

## Pitch

> Run confidential Solana programs. Sensitive inputs never leave Logos's shielded zkVM; Solana only
> ever sees a verified result. We bring confidential computation to Solana's developer ecosystem —
> powered by the one chain whose privacy layer makes it possible.

---

## Next Action

The single most important next step is **Experiment 1** — the interpreter-overhead cycle count. That
one number tells you whether the whole sBPF direction is survivable, and it's a few days of work.
Everything else (the moat, the settlement) only matters if the interpreter economics clear the bar.