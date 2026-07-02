# Part 5: Realtime Proving — The Make-or-Break Piece

> **Series**: 5 of 7 · [Index](./README.md) · [← Previous](./04-from-based-to-eez.md) · [Next →](./06-rollup-adoption-paths.md)
>
> **Prerequisites**: Part 4 (EEZ architecture overview).
>
> **What you'll walk away with**: What "realtime" actually means as a quantitative bar, the cascade failures that follow if proofs are too slow, and why Gnosis is betting on Zisk.

---

## Why did this only become possible in 2026?

By the end of Part 4 you might be wondering:

> "All of this sounds like 2020-vintage technology with new architecture on top. Why did it take until 2026 to actually build?"

Two words: **proof speed**.

Vitalik's preconfirmations paper (one of this series' key references) puts the tension clearly. **All of Based Rollups' virtues — permissionless, censorship-resistant, synchronously composable — rest on one precondition: ZK proofs must be generated within a single L1 block time.** That precondition was fantasy in 2020, barely workable in 2024, and finally mature in 2026.

Let's pin down what "realtime" means before explaining why it's so critical.

---

## 1. What "realtime proving" means

**Realtime = fast enough to fit inside a single L1 block window (~12 seconds).**

Ethereum L1 mints a block every 12 seconds. For synchronous composability to hold, the Composer must complete this entire pipeline within those 12 seconds:

```
L1 block N mined     →  T = 0s
  ↓
Composer receives user intent  →  T = 0.5s
  ↓
Composer simulates L2 execution →  T = 1s
  ↓
Generate ZK proof                →  T = 1 + ??? (this is the crux)
  ↓
Construct + sign L1 transaction  →  T = X + 0.5s
  ↓
Broadcast to L1 mempool          →  T = X + 1s
  ↓
Must arrive before L1 block N+1 (T < 12s)
```

**If proof generation takes 7 seconds**: it fits neatly with room to spare. Your transaction settles in the very next L1 block. **That's what "synchronous" actually means.**

**If it takes 60 seconds**: you miss the next block, and the one after, and the one after. That's not synchronous anymore — that's just a fancier cross-chain bridge.

Zisk's current benchmark is **12 GPUs, ~7 seconds per full Ethereum block**. That number wasn't picked out of a hat — it's the result of pushing the envelope to its practical limit.

---

## 2. What happens if proofs are slow?

A few counterfactuals show just how load-bearing proof speed is.

**Scenario A: 60-second proofs**

- User-facing latency: 60+ seconds. Feels no different from today's cross-chain bridges. Users leave.
- **The L1 state may have moved by then.** Your L2 simulation was based on L1 state at the start. Sixty seconds later, five L1 blocks have passed, and something you depended on (Aave rates, Uniswap prices) may have changed — **simulation invalidated, assertion fails, transaction reverts**.
- High failure rate = frustrated users + Composers eating losses.

**Scenario B: 5-minute proofs**

- "Synchronous" is meaningless.
- Reduced to a more elaborate bridge.
- Everything from Part 2 — the "L1 transaction wrapping L2 execution + assertion" magic — **collapses entirely**. There's no way to stuff a five-minute computation into a twelve-second L1 transaction.

**Scenario C: 12-second proofs (exactly one L1 block)**

- Technically viable, but paper-thin.
- Any hardware jitter, any network hiccup, and you miss the block.
- **Each miss costs another 12 seconds.**
- The experience oscillates. Not commercially viable.

**Scenario D: 3–7 second proofs (EEZ's target)**

- **Same-block completion, reliably.**
- Seconds of headroom against jitter.
- **One L1 proposer can include multiple Composers' cross-chain transactions in the same block.**
- User experience is on par with using Uniswap today.

**Only Scenario D makes "synchronous composability" a real product.**

---

## 3. Proof speed dictates Composer economics

Recall Part 3's Composer role. Its cost structure:

- Hardware: a rack of GPUs.
- L1 gas: paid on every cross-chain transaction.
- Ops: servers, monitoring, engineering.

Its revenue structure:

- User fees.
- MEV extracted from cross-chain arbitrage.
- L2 sequencing tips (where applicable).

**Proof speed directly determines whether this economic model closes:**

- **Slow proofs**: 60 seconds pass, L1 state drifts, assertions fail, transactions revert → **L1 gas paid, refunded to nobody**. Every failure is a net loss. At 5–10% failure rate, the business breaks.
- **Fast proofs**: 7 seconds, L1 state essentially unchanged (same block), failure rate under 1%, fees + MEV comfortably cover costs. Composers compete on price, users get better fills.

**No realtime proofs → no viable Composers → no consumer-grade EEZ.**

---

## 4. Proof speed dictates how many chains you can span

This constraint bites harder for EEZ than for single-chain Based Rollups.

A single-chain Based Rollup only needs to prove one L2 state transition. Seven seconds — fine.

**But EEZ wants L1 and multiple L2s composable in one transaction.** Say a transaction spans L1 + Rollup A + Rollup B + Rollup C:

- Prove Rollup A's state transition.
- Prove Rollup B's state transition.
- Prove Rollup C's state transition.
- Prove the correctness of the cross-chain call orchestration among all three.
- Aggregate everything into one final proof for L1 to verify.

**Do that serially at 7 seconds each and you're at 21+ seconds — already past the L1 block deadline.**

EEZ's answer is **parallelism + recursive aggregation** (see [workshop slide 44](https://docs.google.com/presentation/d/1RREYXg8Sv6cGwAAZ75enjQRt1M_LszgOF4ZJXS-fNLM/edit?slide=id.g3ec2f6daf1e_0_714#slide=id.g3ec2f6daf1e_0_714) — the "EEZ - ZisK proof" diagram):

```
Rollup A proof ─┐
                ├─> Agg ─┐
Rollup B proof ─┘        │
                         ├─> Agg ─> Final ─> PLONK
Rollup C proof ──> Agg ──┘
```

Three rollups prove in parallel (7 seconds each), then recursively aggregate (a few hundred ms per layer). Total pipeline ≈ one proof + O(log N) aggregation overhead.

**And this only works if the individual per-rollup proofs are already fast enough.** If a single rollup takes a minute, no amount of parallelization saves you.

---

## 5. Why Gnosis is betting on Zisk

Zisk shows up repeatedly in the [June 26 EEZ workshop](https://x.com/etheconomiczone/status/2069128518571565163). That's not coincidence — **Zisk is Jordi Baylina's next-generation zkVM, designed specifically for realtime proving.**

Key advantages relative to other zkVMs:

- **RISC-V instruction set**: more general and easier to optimize than a custom zkEVM.
- **Native parallel architecture**: proof jobs decompose into independent chunks.
- **Hardware-friendly**: tuned for GPU clusters; 7-second-per-Ethereum-block is measured, not projected.
- **Aggregation-friendly**: engineered specifically for EEZ's multi-chain aggregation pipeline.

**EEZ and Zisk are co-designed.** EEZ provides the architecture; Zisk supplies the proving engine that makes the architecture work in realtime. Neither can succeed alone.

That's why the workshop insists on "Zisk as the key enabling technology." Without a fast zkVM, EEZ's slides are just an elegant whitepaper. With Zisk, EEZ becomes a system that can ship to mainnet.

---

## 6. The dependency map

```
                          Realtime proving (~7s)
                              │
                              ├─ Makes 12s Based Rollup latency worth it
                              │  (proof fits inside the L1 block window)
                              │
                              ├─ Makes Composer economics viable
                              │  (low failure rate, predictable cost)
                              │
                              ├─ Enables parallel multi-rollup aggregation
                              │  (EEZ's multi-chain reach)
                              │
                              ├─ Delivers Web2-comparable UX
                              │  (click → complete in ~20s)
                              │
                              └─ Turns "synchronous composability" into a shippable product
                                 (the whole point of EEZ)
```

---

## Recap

- Realtime = fast enough to fit inside a 12-second L1 block window. Zisk's benchmark is 12 GPUs, ~7 seconds.
- Slow proofs cascade into system failure: delay → stale L1 state → failed assertions → reverts → Composer losses → ecosystem collapse.
- Multi-chain reach demands parallel aggregation. That only works if individual per-rollup proofs are already fast.
- Zisk is the key enabling technology. EEZ and Zisk are co-designed and mutually necessary.

> **One line**: Realtime proving is EEZ's make-or-break piece — the technical foundation for the word "synchronous." Without it, EEZ degrades into a slightly better cross-chain bridge. With it, EEZ can deliver on the promise of atomic multi-rollup transactions.

**Coming up**: The technology adds up. But today's rollup landscape has 50+ chains — mostly Optimistic Rollups (Arbitrum, Optimism) that don't produce ZK proofs. How do they plug into EEZ? EEZ's pragmatic answer is a three-tier adoption path.

---

**[← Previous](./04-from-based-to-eez.md) · [Index](./README.md) · [Next: How Existing Rollups Plug In →](./06-rollup-adoption-paths.md)**
