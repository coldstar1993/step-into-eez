# Part 2: Synchronous Composability — Where the Magic Happens

> **Series**: 2 of 7 · [Index](./README.md) · [← Previous](./01-based-rollup-foundation.md) · [Next →](./03-composer-and-intents.md)
>
> **Prerequisites**: Part 1 (Based Rollup basics).
>
> **What you'll walk away with**: Why one L1 transaction can wrap a whole slice of L2 execution, what "post-execution assertions" do, and — importantly — a correction to a common misconception about the EVM.

---

## 1. What "synchronous composability" actually means

Here's the exciting part of Based Rollups — the reason they're the path to real cross-chain atomic operations.

### The pain point: cross-chain liquidity is a nightmare today

Say you want to do this:

> **Use your USDC on L2 as collateral to borrow ETH from Aave on L1. In one shot.**

In today's regular-rollup world, this is a mess:

1. First, bridge USDC from L2 back to L1 (either wait through a seven-day challenge period, or pay a third-party bridge).
2. Then use the USDC on L1 as collateral and borrow ETH from Aave.
3. Prices may have moved between steps. Your intended trade might no longer make sense.

**The root cause**: L1 and L2 are separate execution environments. One transaction can only live in one of them, not both.

### The trick: let an L1 transaction swallow an L2 block

Because a Based Rollup's ordering already belongs to L1, we can do something wild:

```
L1 transaction {
    Step 1: Do something on L1 (e.g., pull 100 USDC from my account)
    Step 2: [BEGIN L2 BLOCK]
              Execute a batch of L2 transactions (e.g., swap USDC → WETH on an L2 DEX)
            [END L2 BLOCK]
    Step 3: Take the WETH from L2, deposit it into Aave on L1, borrow DAI
    Step 4: Post-execution assertion — check that DAI borrowed ≥ 200, otherwise revert everything
}
```

**All of this happens inside a single L1 transaction.** Either it all succeeds, or it all rolls back.

### What are "post-execution assertions" for?

They're the safety belt on the whole thing.

Suppose Step 2 goes sideways — the L2 swap suffers heavy slippage and you get 10% less WETH than expected. You don't want the rest of the transaction to proceed (Step 3 won't be profitable). So you tack a line onto the end:

> **"If WETH received < 5, revert."**

That `if` is the post-execution assertion. It's checking the *final aggregate outcome* of the whole cross-chain execution.

- **Assertion passes** → everything sticks. Your collateral, borrow, and state changes are all live.
- **Assertion fails** → `revert`. Step 1 undone. Step 3 undone. And crucially, **that embedded L2 block in Step 2 is also undone** — as if it never happened.

### The bank analogy

Imagine you walk into a bank for a compound transaction:

**Today (regular rollup)**:

- Withdraw $1M cash at HQ (L1).
- Drive two hours to the branch office to deposit into a wealth product (L2).
- Drive two hours back to HQ, present the certificate, apply for a loan.
- Anything goes wrong midway and you're stuck. You've spent money on cabs, you might've been mugged, and there's no undo button.

**Based Rollup mode**:

- You walk into HQ and tell the teller: "I want a compound operation. Use HQ's money to buy the wealth product at the branch, bring the certificate back, use it as collateral for a loan. **If the loan amount is under $500K, cancel the whole thing — money back, certificate cancelled, as if I'd never come in.**"
- The teller hits a button. HQ's system operates the branch's system remotely. All steps run in one shot.
- If the final step fails, **everything unwinds automatically**. You leave as if today never happened.

### What this unlocks

Three immediate applications:

1. **Atomic arbitrage**: ETH sells for $2000 on L1, buys for $2010 on L2. You can buy on L2, sell on L1, and pocket the $10 spread — **no inventory risk** between steps.
2. **Cross-layer flash loans**: Borrow a big pile from Aave on L1, do something profitable on L2, repay Aave — all in one transaction.
3. **Shared liquidity**: Uniswap sits on L1, Curve sits on L2. In a single transaction you can route through both pools at once, and get better price impact than either alone.

### One-line summary

> **A Based Rollup turns an L1 transaction into a Russian doll: an outer L1 shell, an L2 execution slice inside, and an `if` at the end that decides whether the whole doll sticks or vanishes.**

---

## 2. A common misconception to clear up

After reading the above, it's tempting to think:

> "So the L1 EVM runs L1 opcodes *and* L2 state transitions at the same time? But the EVM only runs EVM bytecode — how could it possibly execute L2 code?"

**This is a critical misconception. Let's fix it.**

### The precise version

**The L1 EVM never actually runs L2 code.** The real L2 execution still happens off-chain, in L2 clients. What the L1 EVM does is more subtle — and cleverer.

### Replaying the scenario in detail

Same example: (1) deduct 100 USDC on L1, (2) swap to 0.05 WETH on L2, (3) deposit WETH into Aave on L1.

**Step 1: Someone precomputes the answer off-chain**

Before you send this L1 transaction, a role called the **Composer** (Part 3 explains this in depth) has already done the math off-chain:

- "Assume L2's current state is X. If I run a 100 USDC → WETH swap on it, what's the outcome?"
- The off-chain L2 client says: new state is Y, you get 0.05 WETH.
- The Composer also generates a **validity proof (a ZK proof)** — a small blob of cryptographic evidence that says: "Starting from state X, running this transaction gives state Y. I swear this is correct."

**Step 2: Package the result and the proof into an L1 transaction**

The L1 transaction's calldata looks roughly like:

```
Target: some coordinator contract on L1
calldata: {
    action_1: transfer(user → coordinator, 100 USDC),
    action_2: executeL2Block({
        L2_txs: [swap(100 USDC → 0.05 WETH)],
        expected_new_state_root: 0xABC...,
        validity_proof: 0xDEF...
    }),
    action_3: aave.deposit(0.05 WETH),
    assert: user's Aave balance ≥ 0.05 WETH
}
```

**Step 3: The Rollup Contract on L1 verifies**

When the L1 transaction reaches `action_2`, it's really calling the coordinator contract, which does two things:

1. **Calls the Rollup Contract's `verifyProof(proof, oldState X, newState Y)`.** This is regular EVM bytecode — elliptic-curve math, hashing, all standard stuff. The L1 EVM handles it fine.
2. If verification passes, the Rollup Contract **records** "L2's latest state is now Y" and **surfaces** "user received 0.05 WETH" as an intermediate value for `action_3` to use.

**Step 4: `action_3` picks up the WETH**

The L1 EVM proceeds to `action_3`. It takes the 0.05 WETH that just came out of L2 (which is really just the Rollup Contract's ERC-20 balance for the user — minted after the proof verified) and deposits it into Aave.

**Step 5: The assertion**

`assert` checks whether the user's Aave balance is ≥ 0.05 WETH. If not, `revert`.

### Why the rollback trick works

**A `revert` in the L1 EVM undoes every state change made during this L1 transaction**, including:

- `action_1`'s USDC transfer — undone.
- `action_2`'s "L2 state = Y" write in the Rollup Contract — **also undone**, because that write happened inside this L1 transaction.
- `action_3`'s Aave deposit — undone.

**L2 state can be rolled back because the record "L2 state = Y" lives on L1** (inside the Rollup Contract that we covered in Part 1). Revert the L1 transaction, and the record disappears. Since a Based Rollup's definition is "the L1 contract's state is the L2's official state," erasing the record is the same as L2 never having advanced to state Y.

### A more accurate analogy

You're the CEO of a company (the L1 EVM). You're about to sign a contract. Part of the contract deals with a foreign branch office (L2).

- You don't fly overseas yourself.
- You receive a **notarized report from the branch** documenting what happened there (the validity proof plus new state).
- You write into your contract: "I accept the contents of this report, and here's my portion of the deal."
- If the contract fails for any reason (`revert`), **the whole thing is void** — your portion, and by extension the branch's notarized report loses its effect too.

### The precise formulation

> **The L1 EVM does not execute L2 code. What it does: (1) verify a proof of what happened off-chain on L2, (2) record the L2 state change into the Rollup Contract on L1, (3) use L1 transaction atomicity to bind L1 operations and L2 state changes into a single all-or-nothing unit.**

**"Sharing an execution context" really means "sharing an L1 transaction's success or failure" — not "sharing a single VM instance."**

---

## Recap

- Synchronous composability = one L1 transaction wraps a slice of L2 execution, ended by an assertion. Either everything sticks or everything unwinds.
- Post-execution assertions are the safety belt — let users set "if the outcome doesn't meet my criteria, roll it all back."
- The EVM never actually runs L2 code. L2 execution happens off-chain in L2 clients. The L1 EVM's job is to **verify a ZK proof, record the state change, and use atomicity to bind operations together**.
- The rollback trick works because the L2 state change is recorded on L1. Revert on L1 and the L2 side is automatically undone.

**Coming up**: If a cross-chain transaction is this complex (collecting L2 transactions, generating proofs, constructing L1 calldata), how could a regular user handle it? Answer: they don't. A new role — the **Composer** — does it for them. Users just sign an intent.

---

**[← Previous](./01-based-rollup-foundation.md) · [Index](./README.md) · [Next: Composers and Intents →](./03-composer-and-intents.md)**
