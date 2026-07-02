# Part 4: From Based Rollup to EEZ — Scaling the Magic Across L2s

> **Series**: 4 of 7 · [Index](./README.md) · [← Previous](./03-composer-and-intents.md) · [Next →](./05-realtime-proving.md)
>
> **Prerequisites**: Parts 1–3.
>
> **What you'll walk away with**: EEZ's core ambition, the key architectural components, and why its design is smarter than traditional cross-chain protocols.

---

## 1. EEZ's ambition

At this point you've got the basics:

- Based Rollups let L1 and a single L2 interact atomically inside one transaction.
- Users get access to this through intent signing + Composers.

**So what does EEZ actually add?**

> **It extends Based Rollups' synchronous composability from "L1 ↔ one L2" to "L1 ↔ any number of L2s."**

Concretely, EEZ wants transactions like this to work:

```
One L1 transaction {
    Step 1: Deduct 100 USDC from my L1 account
    Step 2: Swap USDC for WETH on Rollup A (say, zkSync)
    Step 3: Bridge WETH to Rollup B (say, Starknet), deposit into Aave there
    Step 4: Read a price oracle on Rollup C (a specialized oracle rollup)
    Step 5: If the collateral ratio is safe, come back to L1 and mint a stablecoin
    assert: stablecoin received ≥ 200
}
```

**Four chains, one transaction, atomic outcome.**

---

## 2. Core components

If you've seen the [EEZ workshop slides from June 26](https://docs.google.com/presentation/d/1RREYXg8Sv6cGwAAZ75enjQRt1M_LszgOF4ZJXS-fNLM/edit?slide=id.p#slide=id.p), these names should now have a clear home:

### 1. `IRollupContract` (the rollup-side interface)

Every rollup that joins EEZ implements this interface on L1. It's the EEZ-flavored version of the Settlement Contract we covered in Part 1. Through it, the rollup exposes its verifying keys and supported proof systems to the EEZ coordinator.

### 2. The EEZ main contract (Coordinator)

A single L1 contract with an entry function like `postAndVerifyBatch(...)`. This is the orchestrator for all cross-chain atomic transactions — [workshop slides 15–19](https://docs.google.com/presentation/d/1RREYXg8Sv6cGwAAZ75enjQRt1M_LszgOF4ZJXS-fNLM/edit?slide=id.g3ebd545c7e2_0_276#slide=id.g3ebd545c7e2_0_276) show its Solidity guts.

### 3. Execution Table

The complete trace of a cross-chain call, encoded as a table that lives temporarily in L1 state. Each row describes one cross-chain hop (a CALL or a RETURN), carrying the corresponding rollup state transition. **A single ZK proof covers the entire table's correctness in one shot.**

### 4. Proxy Contracts

The technical substrate for cross-chain calls. Every contract on any rollup has a "proxy" on other rollups. The proxy encapsulates all the cross-chain machinery — from the caller's side, it looks and behaves exactly like a local contract.

### 5. Composer market

Same idea as Part 3, but more demanding: it now has to track state across many L2s, produce proofs that span multiple chains, and orchestrate multi-chain execution.

### 6. Realtime Proving

What makes the whole thing feel *instant*. Roughly 12 GPUs can prove a full Ethereum block in ~7 seconds. Any slower and "synchronous composability" becomes "asynchronous with extra steps." **This piece is important enough that Part 5 is dedicated to it.**

---

## 3. Design choices worth noting

**Proof-system agnostic**

EEZ doesn't force every rollup onto the same ZK stack. Rollups can be ZK-based, TEE-based, or even multisig-based. Each rollup declares which proof systems it accepts; the EEZ contract verifies accordingly.

**Rollups keep their sovereignty**

Each rollup defines its own state transition function, governance, and rules. EEZ provides only the cross-chain atomicity primitive — no meddling in internal affairs.

**Permissionless Composers**

Same as single-rollup Based systems: EEZ's Composers are permissionless. No single entity monopolizes cross-chain transaction ordering.

**Orthogonal to preconfirmations**

EEZ doesn't depend on any specific L1 preconfirmation scheme. If they exist, latency improves. If not, the system still works.

---

## 4. The whole world, in one diagram

```
┌─────────────────────────────────────────────────────────┐
│  User (signs only an intent)                            │
└──────────────────────┬──────────────────────────────────┘
                       │ intent + signature
                       ▼
┌─────────────────────────────────────────────────────────┐
│  Composer market (anyone can join)                      │
│  · Simulate multi-chain execution                       │
│  · Assemble the cross-chain plan                        │
│  · Call Prover to generate ZK proof                     │
└──────────────────────┬──────────────────────────────────┘
                       │ L1 transaction (execution table + proof)
                       ▼
┌─────────────────────────────────────────────────────────┐
│  EEZ main contract on L1                                │
│  · Verify the proof                                     │
│  · Update each rollup's state root                      │
│  · Move assets/data across chains via Proxy contracts   │
│  · Assert intent satisfied, otherwise revert all        │
└──────┬────────────┬────────────┬────────────────────────┘
       │            │            │
       ▼            ▼            ▼
   Rollup A     Rollup B     Rollup C
   (each exposes state/verification info via IRollupContract)
```

---

## 5. Why this is only feasible now, in 2026

Three enabling conditions have all landed:

1. **Realtime proving is mature.** ZK proof generation has gone from minutes to seconds. "Same-block cross-chain" is now engineering-tractable.
2. **Intent infrastructure is mature.** ERC-4337, EIP-712, and solver markets have been running on mainnet for a couple of years. Users and wallets are used to the pattern.
3. **Community consensus has formed.** The Ethereum research community broadly agrees that L2 fragmentation is a real problem. Vitalik, Justin Drake, and others have pushed based-rollup thinking into the mainstream.

EEZ stands on all three shoulders and finally turns "multi-rollup synchronous composability" from theory into an engineered system.

---

## Recap

- EEZ's ambition: extend synchronous composability from "L1 ↔ one L2" to "L1 ↔ many L2s."
- Six key components: `IRollupContract`, the EEZ coordinator, execution tables, proxy contracts, the Composer market, and realtime proving.
- Four design choices to notice: proof-system agnostic, rollup sovereignty, permissionless Composers, orthogonal to preconfirmations.
- Three preconditions had to converge for any of this to become buildable in 2026.

**Coming up**: One of those six components — realtime proving — deserves its own deep dive. Why is it the make-or-break piece? Why is 60-seconds-vs-7-seconds a difference in kind, not degree?

---

**[← Previous](./03-composer-and-intents.md) · [Index](./README.md) · [Next: Realtime Proving →](./05-realtime-proving.md)**
