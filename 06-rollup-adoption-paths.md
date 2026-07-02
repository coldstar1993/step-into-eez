# Part 6: How Existing Rollups Plug Into EEZ

> **Series**: 6 of 7 · [Index](./README.md) · [← Previous](./05-realtime-proving.md) · [Next →](./07-eez-significance.md)
>
> **Prerequisites**: Parts 4–5 (architecture + realtime proving).
>
> **What you'll walk away with**: The three tiers of EEZ integration, how Optimistic Rollups reconcile fault proofs with "synchronous," and a realistic timeline for how Arbitrum and Optimism might actually adopt it.

---

## The obvious question

By this point you might be thinking:

> "If EEZ is this good, can Arbitrum and Optimism — today's dominant Optimistic Rollups — actually join? They don't produce ZK proofs at all."

**Short answer**: yes, through one of several paths. EEZ deliberately offers a **spectrum** of integration options, from "barely touch the code" to "deep rebuild." That's one of its most important design decisions as a public infrastructure — **it doesn't force any rollup into major surgery.**

---

## 1. Three adoption paths at a glance

| Path | Rollup-side changes | What you get | Who it suits |
|---|---|---|---|
| **A. Full integration (ZK + realtime proving)** | Major work: add ZK proofs, run realtime proving | Complete synchronous composability | Native ZK rollups, teams willing to rebuild |
| **B. Partial integration (TEE / multisig)** | Moderate work: add an adapter layer | Most of synchronous composability | Optimistic rollups, validator-based rollups |
| **C. Observer mode** | Almost nothing | One-way, delayed composability | Arbitrum and Optimism today |

Let's take them one at a time.

---

## 2. Path A: Full integration (ZK realtime proving)

EEZ's **native** integration model. Fits greenfield rollups or teams willing to rebuild.

**Requirements**:

1. Your rollup can generate ZK validity proofs.
2. Proof generation is fast enough — realtime, roughly 7 seconds per L1 block (see Part 5).
3. Your L1 contract implements the `IRollupContract` interface (see Part 1's Settlement Contract).
4. Your proof system's verifying keys are registered with the EEZ coordinator.

**Who fits**:

- Native zkVM rollups: Zisk-based rollups, Taiko, Linea, Polygon zkEVM, etc.
- New rollups willing to build on Zisk. The workshop slides mention "Chain Zero" as the first fully native EEZ rollup.
- Gnosis Chain itself. Roadmap item 8, "Connecting Gnosis Chain," is exactly this path.

**What you get**: full synchronous composability — every capability from earlier in this series. L1 ↔ your rollup ↔ other EEZ member rollups, all in a single atomic L1 transaction.

---

## 3. Path B: Partial integration (TEE / multisig bridging)

A pragmatic compromise for rollups that can't yet produce ZK proofs.

**The insight**: **"Proof" doesn't have to mean ZK proof.** From workshop slide 5, EEZ explicitly supports:

- ZK-based proofs
- **TEE** (Trusted Execution Environment — Intel SGX, AWS Nitro Enclave, etc.)
- **Multisig** (a committee of trusted validators signing off)

So an Optimistic Rollup can go one of these routes:

1. **Run a TEE sidecar**: run a full rollup node inside Intel SGX or AWS Nitro Enclave. After processing each batch, the enclave produces a hardware-backed attestation — essentially "Intel or AWS is vouching that this batch was executed honestly."
2. **Form a multisig committee**: 15 reputable institutions (Chainlink, Coinbase, Kraken, and so on) each run a node and sign off on each batch. 10 of 15 signatures constitutes a valid attestation.
3. Submit the TEE attestation or multisig data as a "proof" to the EEZ coordinator. EEZ doesn't need to understand the rollup's internals — it just checks the hardware signature or the multisig threshold.

**Who fits**:

- Arbitrum and Optimism — teams that want to join EEZ but aren't ready to add ZK proofs.
- Base, Blast, and other OP Stack rollups. The whole OP Stack ecosystem could layer on one shared EEZ adapter.
- Application-specific rollups that would rather rent security from a committee than build their own ZK stack.

**Security tradeoffs**:

- **TEE**: rests on the Intel hardware and Intel's signing keys not being compromised. SGX has had vulnerabilities historically, but the model is adequate for most financial applications.
- **Multisig**: rests on the committee not colluding above 51%. Similar security model to most bridges in production today.

Both are weaker than native ZK proofs. But they let rollups that can't do ZK still get most of EEZ's benefits. Same posture as accepting Optimistic Rollups' seven-day challenge windows — an explicit, well-understood tradeoff.

---

## 4. Optimistic Rollups and the fault-proof problem

Optimistic Rollups have a built-in feature — **seven-day challenge periods** — that clashes directly with EEZ's "synchronous" (12-second) target.

The fix: **skip fault proofs for EEZ integration; use TEE or multisig instead.** Fault proofs would force a seven-day wait for state confirmation, which is the opposite of what synchronous composability needs.

So a realistic Arbitrum integration looks like this:

```
Arbitrum's core architecture is untouched (fault proofs still exist)
        ↓
An "EEZ adapter" is added in parallel
        ↓
The adapter is operated by a TEE or multisig committee
        ↓
The adapter issues fast attestations for EEZ-related cross-chain transactions
        ↓
EEZ's main contract updates Arbitrum's state based on those fast attestations
```

**Two-track model**:

- **Slow track (fault proofs)**: everyday Arbitrum transactions still get the maximum-security seven-day guarantee.
- **Fast track (TEE / multisig)**: users who want cross-chain atomicity accept the fast track's trust assumptions in exchange for EEZ's capabilities.

Users get an explicit choice. Both tracks coexist.

### What the workshop says

[Slide 7](https://docs.google.com/presentation/d/1RREYXg8Sv6cGwAAZ75enjQRt1M_LszgOF4ZJXS-fNLM/edit?slide=id.g3ec4f8ddcac_0_32#slide=id.g3ec4f8ddcac_0_32) puts it directly:

> "Any centralized rollup can be treated as a base rollup with all its state transitions signed by the centralized sequencer."

Translation: **treat the sequencer's signature as a "proof" and you're in.** That's the lowest possible bar. Optimism's sequencer already signs every batch — plug that signature in as a "1-of-1 multisig" attestation.

Of course, 1-of-1 multisig is a weak security model. In practice Optimism will likely:

- Upgrade to something like 3-of-5 (Optimism Foundation + Coinbase + a few validators), or
- Wait for their fault-proof system to mature and use an asynchronous integration path (high latency, maximum security), or
- Long-term, add ZK proofs — which is already on OP Labs' public roadmap.

---

## 5. Path C: Observer mode (barely any changes)

The lightest possible integration — the rollup does nothing; EEZ reads its state unilaterally.

**How it works**:

The EEZ main contract maintains a "mirror" of external rollup state:

- The coordinator periodically pulls the latest state roots from Arbitrum's / Optimism's L1 contracts.
- Because those rollups' state roots update slowly (fault-proof windows or sequencer submission delays), EEZ always sees a somewhat stale version of their state.
- Users can query "the balance of address X on Arbitrum at time T" from within EEZ.

**What you get**:

- **One-way, asynchronous composability**: your L1 transaction can *read* Arbitrum's old state and use it in a decision, but you can't *write* to Arbitrum.
- Similar to a cross-chain read oracle.

**Who fits**: legacy rollups that don't want to change anything, and scenarios where being read is enough (no need to receive cross-chain writes).

---

## 6. What Optimism and Arbitrum will actually do

The following is a speculative timeline based on the workshop content and current ecosystem dynamics — not an official statement.

**Short term (first 6–12 months after EEZ mainnet)**:

- **Optimism**: Path B via "1-of-1 sequencer signature" for fastest launch, with multisig upgrade planned.
- **Arbitrum**: mostly observing. Offchain Labs has a history of moving cautiously. Likely to start with Path C observer mode, wait for the ecosystem to prove itself, then consider deeper integration.
- **Base**: follows Optimism (it's on OP Stack).

**Medium term (12–24 months)**:

- **Optimism**: upgrades to something like Path B with a 5-of-9 multisig.
- **Arbitrum**: if real demand materializes, likely moves to Path B (TEE or multisig).
- **zkSync / Linea / Polygon zkEVM**: native ZK rollups take Path A and get full synchronous composability.

**Long term (24 months+)**:

- **All major rollups**: as ZK proof costs decline and realtime proving matures, everyone migrates toward Path A.
- End state: **the entire Ethereum L2 landscape is synchronously composable within EEZ**. Cross-chain interaction feels like calling a local contract.

---

## 7. Why the design is elegant

Traditional cross-chain protocols (LayerZero, Wormhole, CCIP) require every rollup to deploy their unified bridge contracts — layering on another trust assumption. EEZ inverts this:

**Sovereignty preserved**

Every rollup keeps its own proof system, state transition function, and governance. EEZ only asks for a "translator" on L1 (an `IRollupContract` implementation) that tells the coordinator how to verify state transitions.

**Marketplace of proof systems**

EEZ doesn't bet on any single proving technology. ZK, TEE, and multisig are parallel options, chosen per rollup and swappable over time. That's how EEZ can accommodate today's rollups (multisig starters) and future rollups (native ZK) under one roof.

**Progressive integration**

Rollups can start at Path C, graduate to Path B, and eventually settle at Path A. Every step improves user experience, but every step is also a valid resting place.

**Fault proofs become optional, not mandatory**

Optimistic Rollups' challenge periods provide maximum security but conflict with synchronous UX. EEZ lets them use the **fast track for cross-chain transactions** and **retain fault proofs for local transactions** — no false choice needed.

---

## Recap

- Three integration paths: A (full ZK), B (TEE/multisig partial), C (observer read-only).
- Optimistic Rollups can integrate. Fast track uses TEE/multisig, slow track keeps fault proofs.
- Realistic timeline: Optimism moves early on Path B, Arbitrum observes first, ZK rollups take Path A immediately, and everyone migrates to Path A over the long run.
- Four design virtues: preserved sovereignty, proof-system marketplace, progressive integration paths, fast/slow dual tracks.

> **One line**: EEZ doesn't force anyone into a big rebuild. It offers a continuum from zero-cost adoption to full native integration — every rollup chooses its own pace and risk profile. That's why [workshop slide 5](https://docs.google.com/presentation/d/1RREYXg8Sv6cGwAAZ75enjQRt1M_LszgOF4ZJXS-fNLM/edit?slide=id.g3ec4ce98da2_0_153#slide=id.g3ec4ce98da2_0_153) highlights "Roll-ups can opt-in, opt-out freely" and "Proof-system agnostic" as core properties.

**Coming up**: The architecture and adoption paths are covered. The final piece pulls back from technical detail to look at the bigger question: what problem does EEZ actually solve, and where does it fit in Ethereum's history?

---

**[← Previous](./05-realtime-proving.md) · [Index](./README.md) · [Next: Why EEZ Matters →](./07-eez-significance.md)**
