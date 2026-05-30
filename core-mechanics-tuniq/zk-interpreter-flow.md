### FLow

Solana bytecode + inputs → your sBPF interpreter runs inside the guest → RISC Zero records the trace of that interpretation → trace becomes polynomials → sealed → Fiat-Shamir challenges → receipt out


## The RISC Zero pipeline (end to end)

Mapped to our sBPF project:

1. **Run + record everything.** Run the guest program (our sBPF interpreter executing Solana bytecode) on RISC-V, recording every register at every cycle → the **execution trace** (a flipbook: one frame per executed instruction).
2. **Trace → polynomials.** Each trace column becomes a polynomial curve. Magic property: different polynomials differ at *almost every* point, so checking a few random points reliably catches any cheat.
3. **Add rule-checks.** Each step must legally follow from the previous one (ADD really added, PC advanced, etc.), encoded as polynomial constraints. Fake one frame → a constraint breaks.
4. **Seal the commitment.** Commit all polynomials via a Merkle tree of hashes (tamper-proof sealed ledger).
5. **Fiat-Shamir challenges.** Hash the commitment to generate random challenge points — no live verifier.
6. **Answer challenges.** Reveal polynomial values at challenge points + Merkle proofs they were in the commitment. (The efficient machinery for this is **FRI** — not yet covered.)
7. **Package the receipt.** Bundle proof + public outputs (the "journal") into a **receipt** — the single verifiable file.
8. **Verification.** Verifier re-runs Fiat-Shamir to confirm honest challenges, checks revealed values satisfy all constraints + match the commitment. Milliseconds, no re-execution.

**Full chain:** Solana bytecode + inputs → sBPF interpreter runs in guest → RISC Zero records the trace → trace → polynomials → sealed → Fiat-Shamir → receipt out → anyone verifies.

---

##  Proving cost — the key engineering insight

**Proving cost scales with instructions you EXECUTE, not bytecode you LOAD.**
- One trace row = one executed instruction = full machine state at one cycle.
- Unexecuted bytecode just sits in the program memory region — loaded but never fetched/decoded/executed → costs ~nothing.

**My idea today (passing only the function signature bytecode instead of the whole binary):**
- Verdict: doesn't give the hoped-for savings — the unexecuted bytecode it cuts *wasn't costing proving cycles anyway.*
- Also can't cleanly slice it: jump offsets are relative to the full loaded image, functions call shared helpers, and Solana bytecode has no clean function boundaries. Slicing breaks the memory model.
- BUT the instinct ("don't pay for work you don't need") is the right foundation for zkVM optimization — just aim it at **executed work**, not bytecode size.

**The real lever — Precompiles / Accelerators:**
Crypto-heavy syscalls explode the trace when interpreted step-by-step.

Example: `sol_sha256`
- **Software path:** RISC-V has no rotate instruction, so each rotate = 3 instructions; SHA-256 does 64 rounds per 64-byte block → roughly **2,000+ trace rows per block**, 30,000+ rows per KB hashed. (Order-of-magnitude; exact counts vary by RISC Zero version.)
- **Accelerator path:** a dedicated, already-proven circuit collapses the whole block-hash into a handful of rows.
- Analogy: long multiplication by hand (every carry is a checkable step) vs. a calculator whose correctness is already certified (one trusted macro-step). Correctness isn't sacrificed — the accelerator circuit is itself proven correct.

**Direct tie to the build — the Syscall Handler is the fork in the road:**
- Naive handler → computes in plain Rust → thousands of trace rows.
- Smart handler → routes to a RISC Zero accelerator → a handful of rows.
- So the syscall handler isn't just "emulate Solana's runtime" — it's the seam where you choose what gets proven the expensive vs. cheap way. Crypto syscalls (`sol_sha256`, `sol_keccak256`, signature checks) matter most.

This connects back to the project doc's optimization techniques (register caching, computed goto): they all shrink the trace.

---

## Open threads for next sessions

- **FRI** — the actual machinery (Step 6) for how STARKs efficiently spot-check polynomials. The "engine room."
- **A concrete trace** — what the "add two numbers" sBPF milestone program looks like as an actual trace table (every row visible). Ties directly to the first project milestone.
- Back to the main build sequence: zkVM host/guest model → ELF loader → memory model → interpreter core → registers → syscalls.

## Earlier groundwork (also covered today)

- **Solana program deployment & "on-chain":** programs are sBPF bytecode stored in a **program account** (`executable: true`, owned by the BPF Loader; ELF bytes live in a linked **ProgramData account**). "On-chain" = that bytecode lives in the validator-replicated account database (consensus state).
- **Account model:** everything is an account; only an account's **owner** can modify its data. BPF Loader owns program accounts; your program owns its state accounts.
- **Execution:** runtime maps 4 virtual memory regions — program text `0x1_`, stack `0x2_`, heap `0x3_`, input buffer `0x4_` — then the sBPF VM runs fetch→decode→execute; `r0` returns 0 (success) or an error.
- **Solana trust model:** validators **re-execute** to verify (economic/social consensus, no proof) — which is exactly the gap our ZK project fills.