# Part 3: User Experience — Intents and Composers

> **Series**: 3 of 7 · [Index](./README.md) · [← Previous](./02-synchronous-composability.md) · [Next →](./04-from-based-to-eez.md)
>
> **Prerequisites**: Parts 1–2.
>
> **What you'll walk away with**: Why regular users don't need to touch any of this complexity, what a Composer actually does, and the role EIP-712 and ERC-4337 play in gluing it all together.

---

## 1. Users never see the complexity

After the last two parts, you might be worried:

> "So users have to collect L2 transactions, generate ZK proofs, construct L1 transactions? That's absurd. Nobody's going to do that."

Right. Nobody expects them to. A new market role handles all of it.

---

## 2. Meet the Composer

The Composer is the most important new actor in the Based Rollup / EEZ ecosystem. From the user's side, it looks like this:

```
1. Open a dApp
2. Type: "swap 100 USDC to WETH, then deposit into Aave"
3. Click "Confirm"
4. Wallet pop-up, sign
5. Done
```

**One signature. That's the entire user job.** Everything else happens backstage.

### What the Composer actually does

1. **Collect intents** from wherever they come in — dApp front-ends, public intent mempools, private order flow.
2. **Simulate execution**: query L2 DEXs for the best route and pricing.
3. **Assemble the L2 block**: bundle this swap (and possibly other users' concurrent transactions) into an L2 block payload.
4. **Trigger proof generation**: hand the execution result to a Prover, which spits out a ZK proof (~7 seconds with a GPU cluster).
5. **Construct the L1 transaction**: package "L1 debit + L2 block data + proof + L1 credit + assertion" into a single L1 transaction.
6. **Submit to L1**, usually through a private channel like Flashbots, to avoid front-running.

**The user pays the Composer a fee — same mental model as paying 1inch today.**

### End-to-end timeline

```
T+0.0s   You click "Confirm" in the dApp
         dApp front-end forwards your intent to a Composer

T+0.1s   Composer receives the intent
         Off-chain simulation across L2 DEX quotes
         → Result: Uniswap V3 on L2 gives 0.0502 WETH

T+0.5s   Composer hands the trace to a Prover
         Prover generates a validity proof on GPU (~7s)

T+7s     Proof lands
         Composer builds the L1 transaction and submits via Flashbots

T+19s    Next L1 slot
         L1 block includes your transaction
         Rollup Contract verifies the proof
         Everything sticks

T+20s    Wallet notifies: "Transaction complete"
         Your Aave balance is up 0.05 WETH
```

**Two user actions total: click and check.**

> **A real Composer project**: Surge is being built specifically to enable "synchronous composability alongside EEZ." See https://x.com/etheconomiczone/status/2052386885876535359

### Anyone can be a Composer

EEZ deliberately keeps this open:

- Large dApps can run their own Composers (e.g., Uniswap's front-end could operate one internally).
- Independent Composer providers can serve many dApps at once.
- Well-capitalized MEV searchers can operate specialized Composers that focus on arbitrage extraction.
- You could run one yourself if you have the technical chops. Most users won't bother.

This preserves **decentralization and censorship resistance**. No single Composer can gatekeep you — swap to another one at will.

---

## 3. What "signing an intent" actually is

When you hit "Confirm," what are you signing? Compare the two approaches.

### The old way: sign a transaction

You sign a precise execution directive:

```json
{
  from: 0xYourAddress,
  to: 0xUniswapRouterAddress,
  data: 0x38ed1739...,  // full calldata for the swap function
  gasPrice: 30 gwei,
  ...
}
```

Everything — route, slippage, gas — is nailed down at signing time. And one transaction lives on one chain. Cross-chain? Forget it.

### The new way: sign an intent

You sign a statement of desired outcome:

```json
{
  user: 0xYourAddress,
  wants:
    - input: up to 100 USDC (on L1)
    - output: at least 0.05 WETH in my Aave account on L1
  deadline: 2026-07-02 15:00:00
  max_fee: 3 USDC
  nonce: 42
}
```

"Reach this outcome. How you get there and who does it — I don't care."

---

## 4. Three technical pillars

For the intent-signing → on-chain-execution flow to actually work, three pieces need to be in place:

**Pillar 1: EIP-712 structured signatures**

What you sign isn't raw bytes — it's a structured payload following the EIP-712 spec. This lets your wallet render **human-readable content** ("swap up to 100 USDC for at least 0.05 WETH") instead of a hex blob. It also prevents the same signature from being replayed on a different contract or chain.

**Pillar 2: ERC-4337 account abstraction**

Traditionally your account is an EOA — your private key has to sign and send every transaction. ERC-4337 lets you sign an *operation description* and delegate the actual sending to a bundler (in EEZ, that's the Composer). Two big wins:

- You don't need ETH on L1 to pay gas — the Composer fronts it and deducts from what you're sending.
- Wallets can support social recovery, multisig, biometrics, and richer patterns.

**Pillar 3: The solver market**

A Composer is fundamentally a solver — a market participant that finds the best execution path for a user's intent. This model has been running on Ethereum for years (CoWSwap, 1inch Fusion, UniswapX). EEZ extends it to **cross-chain intents**.

---

## 5. How is user safety guaranteed?

Signing an intent doesn't require blind trust in the Composer. **The intent's constraints are enforced by L1 contract code**:

```solidity
function executeIntent(Intent memory intent, bytes memory sig, Plan memory plan) {
    // 1. Verify signature
    require(ecrecover(hash(intent), sig) == intent.user, "signature mismatch");

    // 2. Verify not expired
    require(block.timestamp <= intent.deadline, "intent expired");

    // 3. Anti-replay
    require(!usedIntents[hash(intent)], "intent already used");
    usedIntents[hash(intent)] = true;

    // 4. Pull funds — authorized by the intent signature
    IERC20(USDC).transferFrom(intent.user, address(this), intent.maxInput);

    // 5. Execute the Composer's plan
    executePlan(plan);

    // 6. Final assertion: was the intent satisfied?
    require(checkIntentSatisfied(intent, plan.result), "intent not satisfied");
}
```

**If the Composer tries to cheat**:

- Shortchange your WETH? → step 6 fails → revert → no loss to you.
- Charge more than the max fee? → assertion fails → revert.
- Use your authorization for something else? → revert.

**Your funds are protected by contract code, not by trust in any Composer.**

---

## 6. The Uber analogy

The structure resembles ride-hailing:

| Uber | Cross-chain intent |
|---|---|
| "Take me to the airport" | "Get me 0.05 WETH into Aave" |
| System matches a driver | Composer market competes for the order |
| Driver's route is their business | Composer's L1/L2 routing is their business |
| Didn't reach the airport? Cancel, no charge | Intent not satisfied? Assertion reverts, no loss |
| Price cap protection | maxFee protection |

---

## Recap

- Users never touch L2 transactions or proofs. Composers handle the grunt work.
- Users sign an **intent** — a description of the outcome they want, not a specific route.
- The three enabling pillars: EIP-712 (readable signatures), ERC-4337 (Composers can send transactions on your behalf), solver markets (Composers compete on price).
- Safety comes from contract-enforced constraints, not trust.
- Anyone can be a Composer. If one misbehaves, users route to another.

**Coming up**: So far we've discussed L1 ↔ one L2. EEZ's real ambition is L1 ↔ any number of L2s — one transaction atomically touching four different chains. That takes new architecture: the EEZ coordinator contract, execution tables, proxy contracts.

---

**[← Previous](./02-synchronous-composability.md) · [Index](./README.md) · [Next: From Based Rollup to EEZ →](./04-from-based-to-eez.md)**
