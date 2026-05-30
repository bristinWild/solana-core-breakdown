# Research Summary — Day 1

Topics: Arcium, MPC vs ZK, zero-knowledge proofs, STARKs vs SNARKs, Fiat-Shamir, and the RISC Zero pipeline.

> Context: this all feeds the main project — building an sBPF interpreter inside the RISC Zero zkVM so Solana programs can run with ZK-proved correctness.

> Note on "MCP": in our conversation this referred to **MPC = Multi-Party Computation** (Arcium's core tech), not the Model Context Protocol. Summarized as MPC below.

---

## 1. Arcium — and the MPC vs ZK distinction

**Key correction from today:** Arcium is primarily an **MPC network, not a ZK-proof system.** They solve a similar *business* problem to our project (private/verifiable compute, then a trusted on-chain result) but with an opposite cryptographic engine.

- Arcium = decentralized encrypted compute using **Multi-Party Computation**. Data is split into secret shares across "Arx nodes"; no single node ever sees the full input. Privacy holds as long as one node is honest. Trust comes from **spreading the work out**.
- They use ZK only in a *supporting* role (e.g. proving a transaction is valid without revealing details), not as the main engine.

**MPC analogy (three accountants):** To compute an average salary without anyone revealing their salary, each person tears their number into random scraps that mean nothing alone, hands one scrap to each accountant, and only the *combined* partial sums reveal the final average. No accountant ever held a real number.

**What actually transfers to our project:** NOT their cryptography — their **architecture pattern**:
> Off-chain confidential/heavy compute → return only a verifiable result on-chain. Keep the on-chain part simple and auditable.

- Solana = "open kitchen" (every validator re-executes everything).
- Arcium & our project = "sealed kitchen" (do the work aside, send out the finished plate + a guarantee).
- Difference: our guarantee is a **STARK proof (math)**; theirs is **MPC consensus (distributed trust)**.

---

## 2. Zero-knowledge proofs — core mental model

**Roles:**
- **Prover** = creates the proof (does the work, e.g. runs the sBPF program).
- **Verifier** = checks the proof (fast, no re-execution).

**Does the verifier supervise the prover? No — the math makes cheating impossible instead.**
A dishonest prover isn't "caught breaking a rule"; they simply **cannot produce a proof that passes**. So "did the prover follow the protocol?" is answered automatically by "does the proof verify?"

**The commit-then-challenge trick (book analogy):**
1. Someone claims they read a 1,000-page book.
2. They must **commit to their full answer sheet first** (sealed, tamper-proof).
3. *Then* you flip to a few **random** pages and quiz them.
4. A faker can't survive random spot-checks they couldn't predict. ~20 random checks → essentially zero chance of cheating.

Cheating probability in real systems ≈ 1 in 2^128 (won't happen before the heat death of the universe).

**Where's Waldo analogy (the "zero-knowledge" part):** prove you found Waldo by showing him through a tiny hole in a big cover sheet — proves he exists / you found him, without revealing *where* on the page.

---

## 3. STARKs vs SNARKs

Both are ZK proof systems with different trade-offs. **RISC Zero uses STARKs.**

Name decoding:
- **SNARK** = Succinct Non-interactive ARgument of Knowledge
- **STARK** = Scalable Transparent ARgument of Knowledge

**Jigsaw puzzle analogy (proving you solved a 10,000-piece puzzle):**
- **SNARK = sealed photograph.** A special camera shrinks the finished puzzle to a postage stamp — instant to check. BUT the camera came from a "secret factory" (trusted setup); if someone kept the secret blueprint ("toxic waste"), they could forge proofs.
- **STARK = public spot-check with sealed answers.** No factory. You seal a ledger of every piece, then a friend rolls dice *in plain sight* to pick which pieces to check. Nothing secret exists to steal — but the proof is a fat dossier, not a stamp.

| Dimension | SNARK (sealed photo) | STARK (public spot-check) |
|---|---|---|
| Proof size | Tiny (postage stamp) | Chunky (a dossier, 10s–100s KB) |
| Trusted setup | Yes — "secret factory" (risky) | **None — fully transparent** |
| Quantum-safe | Often no (elliptic-curve pairings) | **Yes (hash-based only)** |
| Big computations | Strains | **Scales gracefully** |

**Why STARKs fit our project:** transparency (no setup) + quantum resistance come "for free," and they scale well for huge computations — exactly the zkVM case (potentially millions of interpreted sBPF instructions). The cost we accept is larger proof size.

---

## 4. Fiat-Shamir — removing the live verifier

**Problem:** the spot-check story assumes a live friend rolling dice (interactive). We want a single proof file checkable later by anyone, with nobody around.

**The trick:** replace the live dice-roller with the **proof itself**. Take your own sealed commitment, run it through a hash function (a "shredder-blender") to get an unpredictable number, and use *that* to choose the spot-check challenges.

**Why it's safe:** hashes are wildly sensitive — change one piece and the output is completely different. You can't rig your puzzle to get easy challenges, because you'd need to know the final hash *before* sealing the puzzle, which is impossible (the hash is computed *from* the sealed puzzle). Chicken-and-egg trap only an honest prover can satisfy.

**Analogy:** the dice are *made by melting down the very puzzle you just sealed* — you can't predict the rolls until after you've committed everything.

**Result:** proof becomes a single non-interactive file = sealed commitment + answers to the challenges its own hash generated. This is the "N" / "non-interactive" in SNARK and STARK.

---

