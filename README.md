<!-- solve-problem:e46d4b45-c3ed-479d-baef-a9bfe69864ac -->
# Succinct adversarial DAG pruning proofs

A [Tangle Solve problem](https://solve.tangle.tools/problems/e46d4b45-c3ed-479d-baef-a9bfe69864ac).

## What is the problem?

In Proof-of-Work DAG-based ledgers (e.g., PHANTOM/GHOSTDAG), blocks may reference multiple predecessors, and the consensus rule is: (i) order the DAG via a coloring/fork-choice rule (GHOSTDAG orders blocks by a greedy "blue set" selection parameterized by an anti-size parameter *k*), and (ii) accept transactions and count "work" only on blocks in the chosen chain/order. Nodes periodically **prune** — discard old block data (headers, transactions, UTXO commitments) — keeping only recent history plus a checkpoint.

The problem: construct a **succinct, non-interactive proof** that pruning was performed *honestly*, i.e., a proof binding (a) the pruned state (UTXO set / transaction-validity state), (b) the accumulated work of the surviving canonical structure, and (c) the correctness of the fork-choice outcome at the prune point — without any trusted committee or assumed-honest checkpoint server. A light client holding only O(log n) or O(1) data should be able to verify that the pruned digest it is given is the unique honest continuation of the prefix it previously knew.

Formal setting: a block DAG *D* = (V, E) with each block carrying PoW, a hash pointer set, and transactions. Pruning produces a digest σ = Prune(D). Goal: an argument system proving "∃ D consistent with my prior commitment σ₀, with total difficulty ≥ that of any conflicting DAG, whose GHOSTDAG ordering applied to the transaction set yields exactly the state committed in σ."

## Why does it matter?

Solving this lets full nodes in DAG ledgers run with bounded storage while light clients retain Bitcoin-grade security guarantees; without it, DAG histories (which grow faster than chains by design) make node operation strictly more expensive than chains, undermining the throughput motivation for DAGs in the first place. It is the missing piece for "super-light" clients in high-throughput PoW systems — a direct generalization of FlyClient's chain-type result. Failure to solve it means DAG ledgers either require trusted checkpoints (a centralization/safety regression) or unbounded historical storage.

## What's already known?

- **FlyClient (Bünz, Kalfa, Goldfeder, Juels 2020)** solves the chain case: a probabilistic proof that a prover's chain is the heaviest, using Merkle Mountain Ranges over random-sampled blocks sampled by a Fiat-Shamir-derived seed, plus a chunked NIPoW-style argument. This is the baseline to generalize; it crucially exploits total order (chain = path), so the sampling game "sample a block at height *h*, check it extends the prior" has no DAG analogue yet.
- **Non-Interactive Proofs of Proof-of-Work (Kiayias, Miller, Zindros 2017-2019, "NIPoPoWs")** give succinct chain prefixes; again linear-order-only.
- **GHOSTDAG (Sompolinsky & Zohar 2021; Sompolinsky et al. 2023)** gives the fork-choice rule over DAGs — the blue-set coloring — but its papers do not address light clients or pruning; Kaspa's production system uses periodic pruning *with trusted-ish archival assumptions* (verify current state of Kaspa pruning claims before citing in a paper).
- **Authenticated data structures / streaming verification** (e.g., verified streaming via Merkle trees, authenticated graphs) give tools for committing to graph states, but not to the *game-theoretic* property (heaviest-DAG dominance).

No published construction is known that simultaneously proves work accounting and fork-choice correctness over a pruned DAG.

## What would count as solving it?

1. **Construction:** A non-interactive argument (in the random oracle model) between an honest prover (a pruning full node) and a verifier with O(polylog n) state, such that: (a) completeness — honest pruning yields accepting proofs; (b) soundness — any accepting proof for a digest σ implies σ corresponds to a DAG whose GHOSTDAG-ordered prefix dominates (in appropriately generalized "heaviest" terms) any conflicting DAG extending the client's prior commitment, except with negligible probability given honest majority of mining power.
2. **Proof of the security reduction** to the underlying PoW and fork-choice properties (as FlyClient §4-5 does for chains).
3. **Sizes:** proof size and verifier time polylogarithmic in history length; prover overhead linear(ish) per prune.
4. Partial credit: solve only the *work accounting* part (a FlyClient-style sampling game on DAG block counts), or only *transaction-state* binding (authenticated UTXO commitment + validity under the blue-set ordering).

## How to start

1. Read FlyClient (ePrint 2019/226), especially the sampling game and the "committee"-free security proof; then read GHOSTDAG's formal definitions of blue sets and the ordering ≤_k, since any proof must argue dominance of one coloring over another.
2. Formalize "heaviest DAG" as the comparison predicate the sampling game must verify — the key open sub-question: GHOSTDAG fork choice is *not* simply summed work over the DAG; determine whether the recursive blue-set structure admits a monotone, locally-checkable certificate (e.g., per-block blue-count commitments).
3. Design the digest: a Merkle commitment over a "prune-point header" containing (UTXO root, blue-block work root, DAG-structure commitment over the last *w* layers); then a challenge-response game where the client samples blocks and the prover shows the block's color and its contribution to work.
4. Main obstacle: the adversary can present a *different coloring* of the same DAG subset; you must show conflicting colorings imply detectably divergent work or a violated anticone bound — this is where the *k* parameter's robustness (GHOSTDAG's probability bounds) must be invoked.

## Key references

- Bünz, Kalfa, Goldfeder, Juels. *FlyClient: Super-Light Clients for Cryptocurrencies.* IACR ePrint 2019/226. https://eprint.iacr.org/2019/226 — Builds MMR-based random-sampling NIPoWs proving a chain is the longest/heaviest with logarithmic communication; the direct predecessor whose machinery must be generalized.
- Sompolinsky, Lewenberg, Zohar et al. *PHANTOM and GHOSTDAG* (ACM TAAS / the DOI 10.1145/3479722.3480990 corresponds to a 2021-2023 PHANTOM-family publication — verify exact venue). Defines the k-cluster coloring that yields a total order on the block DAG and proves robustness under attacker hash-rate bounds.
- Kiayias, Miller, Zindros. *Non-Interactive Proofs of Proof-of-Work.* (CVCBT/Financial Crypto era, ~2017-2021, verify venue) — NIPoPoWs for succinct chain prefixes; useful for the "interlink" commitment technique.
- (verify) Kaspa protocol documentation on periodic pruning — engineering precedent showing what needs to be made trustless.

## Warnings

- The census record labels this "engineering," but the core blocker is a *soundness definition problem*: it is genuinely unclear what "pruning preserves fork choice" means as a provable predicate when fork choice is a global recursive function of the DAG. Fix the predicate before writing any protocol.
- Do not assume GHOSTDAG ordering is locally checkable — it depends on anticone relations that pruning may delete; you may need to commit to anticone witnesses.
- Chain-based NIPoPoW/FlyClient security proofs lean on "longest chain" monotonicity; DAG heaviest-structure comparisons are strictly messier and known proof templates do not transfer directly.
- The DOI given in the census record may not resolve to the exact GHOSTDAG paper you expect — verify before citing.

---
*`BCGT-000258` · flagged for review*

## Acceptance guidance

1. **Construction:** A non-interactive argument (in the random oracle model) between an honest prover (a pruning full node) and a verifier with O(polylog n) state, such that: (a) completeness — honest pruning yields accepting proofs; (b) soundness — any accepting proof for a digest σ implies σ corresponds to a DAG whose GHOSTDAG-ordered prefix dominates (in appropriately generalized "heaviest" terms) any conflicting DAG extending the client's prior commitment, except with negligible probability given honest majority of mining power.
2. **Proof of the security reduction** to the underlying PoW and fork-choice properties (as FlyClient §4-5 does for chains).
3. **Sizes:** proof size and verifier time polylogarithmic in history length; prover overhead linear(ish) per prune.
4. Partial credit: solve only the *work accounting* part (a FlyClient-style sampling game on DAG block counts), or only *transaction-state* binding (authenticated UTXO commitment + validity under the blue-set ordering).

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/e46d4b45-c3ed-479d-baef-a9bfe69864ac?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000258.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000258)
