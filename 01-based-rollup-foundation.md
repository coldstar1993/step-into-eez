# Part 1: The Foundation — Who Controls Sequencing?

> **Series**: 1 of 7 · [Index](./README.md) · [Next →](./02-synchronous-composability.md)
>
> **Prerequisites**: Basic familiarity with Ethereum (L1, L2, smart contracts).
>
> **What you'll walk away with**: The core distinction between a regular rollup and a based rollup, why L2 blocks don't need their own chain structure, and what really sits inside a rollup's L1 contract.

---

## 1. Two schools of rollup design

Everything about Based Rollups traces back to one question: **who gets to decide the order of transactions?**

### The delivery company analogy

Picture a delivery company sorting packages, loading trucks, and dispatching them. Someone has to decide which package goes on which truck and in what order — that's sequencing. Rollups answer this question in two very different ways.

**School A — Regular rollups (Arbitrum, Optimism, zkSync, and so on)**

These rollups hire their own dispatcher, called a **sequencer**. The sequencer orders every L2 transaction, then periodically ships batched results back to headquarters (Ethereum L1).

- **Upside**: The dispatcher works on-site. Everything's fast — you submit a transaction, you get a receipt in seconds.
- **Downside**: The dispatcher is centralized. In principle, they can jump the queue, take breaks, or refuse to serve certain customers.

**School B — Based Rollups**

Skip the dispatcher entirely. Let headquarters (Ethereum mainnet block producers) do the sequencing directly.

- **Upside**: No centralized operator. Permissionless — anyone can push a transaction in. Censorship resistance is inherited straight from Ethereum.
- **Downside**: Slow. You wait a full Ethereum block (~12 seconds) before your L2 transaction has an official position.

### Where the name comes from

"Based" is short for **L1-based sequencing** — a phrase coined by Ethereum researcher Justin Drake. It's meant to sound both technical and slightly cheeky.

### Side-by-side comparison

| | Regular Rollup | Based Rollup |
|---|---|---|
| Latency | Fast (sub-second preconfirmations) | Slow (~12s L1 block time) |
| Permissionless | Weak (sequencer-dependent) | Strong |
| Censorship resistance | Weak | Strong (same level as L1) |
| **Synchronous composability with L1** | ❌ | ✅ |
| MEV ownership | Sequencer | L1 proposer |

**That last row is what the rest of this series is about.** Only Based Rollups can compose synchronously with L1. We'll unpack what that means starting in Part 2.

---

## 2. The core insight: L1's order is L2's order

Here's a question beginners often ask:

> "If L1 decides the order of L2 transactions, do L2 blocks still need to reference their parent block's hash? Without that chain structure, how does L2 hold together?"

Good question — it points right at what makes Based Rollups counterintuitive.

### L2 blocks aren't an independent chain — they're a derivation

The L2 blocks in a Based Rollup **aren't produced by L2 nodes voting on anything**. They're the output of a function:

```
L2 state_{n+1} = f(L2 state_n, data in this L1 transaction)
```

So the sequence goes:

1. Someone submits a "batch transaction" to L1.
2. L1 mines a block that includes it.
3. Every Based Rollup client, seeing that L1 transaction, computes locally: "OK, L2 has moved to its next state."

**There's no L2-side voting, no L2 block gossip, none of that.** "L2 block #100" is just "whatever state emerges after L1 has included the 100th batch transaction."

### So where's the chain?

**L1's block order is L2's block order.**

- L1 is a proper chain — each block references its parent's hash.
- Once a Based Rollup batch is included in an L1 block, its position is nailed down by L1's hash chain.
- L2 clients read L1 sequentially: the first batch produces L2 block #1, the second produces #2, and so on.
- **L2 block #2's parent is L2 block #1**, but this parent-child relationship doesn't require L2 to sign anything — it falls out of L1's ordering for free.

### DVD vs. Netflix

Think of watching a TV series:

- **Regular Rollup** = Netflix. Every episode carries "Previously on..." metadata inside itself. Each episode declares what comes before it.
- **Based Rollup** = DVD. Individual episodes don't say what number they are — but the disc's table of contents lists "Episode 1 is on track A, Episode 2 on track B, Episode 3 on track C." Play them in order and it all works.

**The table of contents is L1. The tracks are the transactions inside L1 blocks.** If someone wanted to reorder the episodes, they'd have to rewrite the disc's index — but that index is protected by Ethereum's consensus. Not going to happen.

### The mental shift

> **Based Rollups' core insight: since L1 is already a censorship-resistant, decentralized chain, why should L2 build one from scratch? Piggyback on L1's ordering and drop the complexity entirely.**

That's exactly what "based" means — L2's chain structure is *based on* L1, not standalone.

---

## 3. What actually lives on L1? The Settlement Contract

If L2 state is derived from L1, then L1 needs some kind of anchor to hold that state. Every rollup has one: the **Settlement Contract** (sometimes called the Rollup Contract).

### Four jobs it does

**Job 1: Store L2's state commitment (state root)**

Inside the contract, something like:

```solidity
bytes32 public latestStateRoot;
uint256 public latestBlockNumber;
```

**The state root here is the single source of truth for L2.** Whatever L2 clients compute locally is only "official" if it matches this value on L1.

**Job 2: Accept new L2 block submissions**

A function anyone can call (or only the sequencer, depending on the rollup):

```solidity
function submitBlock(
    bytes calldata l2BlockData,      // the L2 block payload
    bytes32 newStateRoot,            // the claimed new state
    bytes calldata validityProof     // a ZK proof, if this is a ZK rollup
) external {
    require(verifyProof(validityProof, latestStateRoot, newStateRoot, l2BlockData));
    latestStateRoot = newStateRoot;
    latestBlockNumber++;
    emit BlockSubmitted(latestBlockNumber, newStateRoot);
}
```

Once this call succeeds, L2 has officially advanced.

**Job 3: Run the asset bridge**

Deposits from L1 to L2 and withdrawals from L2 to L1 all flow through this contract (or a bridge contract paired with it):

- **Deposit**: You send ETH or an ERC-20 to the contract. The contract records "address X should have Y balance on L2." L2 clients see that L1 record and mint the corresponding asset for you on L2.
- **Withdrawal**: You burn the asset on L2 → produce a proof → the L1 contract verifies it → funds return to your L1 account.

**All L2 assets are, at their core, held in this contract.** Moving funds around on L2 is really just moving accounting entries against the custody sitting here.

**Job 4: Verify proofs**

- **ZK Rollups**: A `verifyProof` function runs elliptic-curve math to check a ZK proof. If it passes, the state transition is legal.
- **Optimistic Rollups**: A fraud-proof mechanism. Assume the submitted state is correct, but give people seven days to prove it wrong. If nobody objects, it stands.

### What makes a Based Rollup's contract special

Structurally, it's almost identical. **The one meaningful difference is who can call `submitBlock`.**

| | Regular Rollup | Based Rollup |
|---|---|---|
| Who can call `submitBlock` | Only the centralized sequencer | **Anyone** |
| Ordering logic | Sequencer decides | Falls out of L1 block order |
| L2 mempool | Sequencer's private pool | Usually visible in L1 mempool |

**A more radical variant** — and this is what EEZ pushes toward — skips `submitBlock` altogether. The Rollup Contract watches every L1 block's calldata directly. If it spots valid L2 transaction data, it derives the new L2 state on the spot. That means one L1 transaction can carry a chunk of L2 execution as part of its own flow. That's the setup for what comes next.

---

## Recap

- Rollups split into two schools: regular ones use a sequencer (fast but centralized), Based Rollups use L1 itself (slow but permissionless).
- Based Rollups' key insight: L2 blocks aren't a separate chain — they're derived from L1's transaction order. L1's ordering *is* L2's ordering.
- Every rollup has a Settlement Contract on L1 that does four things: hold state root, accept block submissions, run the bridge, verify proofs.
- What's different about Based Rollups: `submitBlock` is permissionless. More aggressive designs let a single L1 transaction embed a slice of L2 execution.

**Coming up**: Now that an L1 transaction can carry L2 execution inside it, what does that enable? Part 2 covers the magic — how one atomic L1 transaction can do "spend USDC on L1, swap to WETH on L2, deposit into Aave on L1" as a single, all-or-nothing operation.

---

**[← Index](./README.md) · [Next: Synchronous Composability →](./02-synchronous-composability.md)**
