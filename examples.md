# Usage Examples

The following examples outline high-level constructions made possible by this proposal. The term **covenant** is used to describe a Bitcoin Cash contract that inspects the transaction that spends it, enforcing constraints – e.g. a contract that requires user to send an equal number of satoshis back to the same contract (such that the contract maintains its own "balance").

## Identity Tokens

Prior to this proposal, Bitcoin Cash contracts were limited to interacting with static identities, usually defined by one or more public keys. A contract might allow a particular identity to spend funds under certain conditions (e.g. after a fixed delay), but if an identity needs to update its public key(s), the migration would require that the contract be re-created and the funds moved.

This proposal enables contracts to instead interact with **identity tokens** – non-fungible tokens that prove control of a represented identity. Identity tokens can be moved independently of the contracts that verify them, allowing greater flexibility and control over the identities within contracts, e.g. a user may upgrade to a two-factor (multisig) wallet, a company may rotate keys after a security breach, or an organization may add or remove individuals with signing authority. With identity tokens, contracts can be designed to allow identities to perform these updates without re-creating the contract, reducing the cost and complexity of coordinating with other contract participants.

## Covenant-Tracking Identity Tokens

A covenant can be associated with a **"tracking" identity token**, requiring that spends always re-associate the identity token with the covenant.

Beyond simplifying logic for clients to safely locate and interact with the covenant, such a tracking token offers an impersonation-proof strategy for other contracts to authenticate a particular covenant instance. This primitive enables covenants to design **public interfaces**, paths of operation intended for other contracts (which may themselves be designed and deployed after the creation of the original covenant).

Because token category IDs can be known prior to their creation, it is straightforward to create ecosystems of contracts that are mutually-aware of each other's tracking token category ID(s).

Notably, tracking tokens also allow for a significant contract-size and application-layer optimization: a covenant's internal state can be written to its tracking token's `commitment`, **allowing the locking bytecode of covenants to remain unchanged across transactions**.

## Depository Child Covenants

Given the existence of [covenant-tracking identity tokens](#covenant-tracking-identity-tokens), it is trivial to develop **depository child covenants** – covenants that hold a non-fungible token and/or an amount of fungible tokens on behalf of a parent covenant. By requiring that the depository covenant be spent only in transactions with the parent covenant, such depository covenants can allow a parent covenant to hold and manipulate token portfolios with an unlimited quantity of fungible or non-fungible tokens (despite the limitation to [one prefix codepoint per output](rationale.md#one-prefix-codepoint-per-output)).

## Voting with Fungible Tokens

This proposal allows decentralized organization-coordinating covenants to hold votes using fungible tokens (without pausing the ability to transfer, divide, or merge fungible voting token outputs during the voting period).

In short, covenants can migrate to a new token category over a voting period – shareholders trade in amounts of "pre-vote" shares, receiving "post-vote" shares, and incrementing their chosen result by the `amount` of share-votes cast.

This construction reveals additional consensus strategies for decentralized organizations:

- **Vote-dependent, post-vote token categories** – covenants that rely on absolute consensus – like certain sidechain bridges (which, e.g. must come to a consensus on an aggregate withdrawal transaction per withdrawal period and then penalize or confiscate dishonest shares) – can issue different categories of post-vote tokens based on the vote cast. This allows for transfer and trading of post-vote tokens even before the voting period ends. When different voting outcomes impact the value of voting tokens (after the voting period ends), such differences will **immediately appear in market prices of post-vote tokens**. This observation presents further consensus strategies employing hedging, prediction markets, synthetic assets, etc.
- **Covenant spin-offs** – in cases where a covenant-based organization plans a spin-off (e.g. a significant sidechain fork), covenant participants can be allowed to select between receiving shares in the new covenant or receiving an alternative compensation (e.g. a one-time BCH payout or additional shares in the non-forking covenant).

To support simultaneous shareholder voting on multiple questions or to avoid the need for a token category ID migration, votes can be performed by submitting shares to trustless covenants implementing other strategies:

- **voting receipts** – rather than issuing a post-vote fungible token, vote-accepting covenants can accept fungible tokens of the original category as deposits, issuing non-fungible depository "receipt" tokens that entitle the holder to reclaim the deposited amount of original fungible tokens after the voting period ends. Voting receipts can also retain information about the vote cast such that the vote-accepting covenant can allow voters to "uncast" their votes and immediately withdraw their deposited tokens (e.g. to divest).
- **single-use voting tokens** – likewise, vote-accepting covenants can also issue multiple batches of single-use tokens in exchange for deposits of the original category. Each set of single-use tokens can be required by different vote-accepting covenants, and redeeming the deposited tokens of the original category may require either a depository receipt (e.g. after all voting periods end) or an equal amount of each issued single-use voting token (in uses cases where votes may be uncast before the voting period ends).

Finally, note that the above discussion focuses on coordinating on-chain voting for on-chain entities; off-chain entities can always conduct votes using off-chain coordination strategies, e.g. by accepting votes via [Bitauth signature](https://github.com/bitauth/bitauth-cli) from any output holding fungible tokens as of a previously-announced block height or [Median Time Past](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki).

### Sealed Voting

**"Sealed" voting** – in which the contents of votes are unknown until after all votes are cast – is immediately possible with this proposal.

Voters cast a **sealed vote** – a message containing 1) the number of share-votes cast and 2) a hash of their vote concatenated with a [salt](<https://en.wikipedia.org/wiki/Salt_(cryptography)>) – by minting a **sealed vote NFT** from the vote-taking covenant. Once the voting period has ended, each participant can re-submit all sealed vote NFTs to the covenant, aggregating results in another part of the covenant's state.

This basic construction can be augmented for various use cases:

- **voting quorum** – requiring some minimum percentage of sealed votes before a voting period ends.
- **unsealing quorum** – requiring some minimum percentage of sealed votes to be unsealed before vote results can be considered final.
- **sealing deposits** – requiring voters to submit a deposit with sealed votes that can be retrieved by later unsealing the vote.
- **enforced vote secrecy** – allowing voters to submit sealed "unsealing proofs", proving that another voter divulged their unsealed vote prior to the end of the voting period. Such proofs could reward the submitter at the expense of the prematurely-unsealing voter<sup>1</sup>, frustrating attempts to coordinate malicious voting blocs. (A strategy [developed for Truthcoin](https://bitcoinhivemind.com/papers/truthcoin-whitepaper.pdf).)

<details>
<summary>Notes</summary>

1. This can be accomplished by requiring each sealed vote NFT to be placed in a time-delayed "unsealing covenant" that accepts such proofs. If the penalized voter can't be relied upon to re-submit a sealed vote NFT during such an enforcement period, the vote-accepting covenant must maintain sealed votes in its internal state. For example, a "ballot box" merkle tree can contain at least as many empty leaves as outstanding shares. Sealed votes are submitted by replacing an empty leaf in the ballot box (as demonstrated in [CashTokens v0](https://gist.github.com/bitjson/a440232cebba8f0b2b6b9aa5db1fdb37)), and the process is reversed to prove the contents of sealed votes, enforce penalties, and aggregate results.

</details>

## Multithreaded Covenants

While most account-based contract models are globally-ordered by miners (e.g. Ethereum), the highly-parallel, UTXO model employed by Bitcoin Cash requires that **contract users determine transaction order**. This has significant scaling advantages – transactions can be validated in parallel, often before their confirming block is received, [enabling validation of 25,000 tx/second on modest hardware (as of 2020)](https://read.cash/@TomZ/scaling-bitcoin-cash-be8344a6). However, this model requires contract developers to account for transaction-order contention.

Transaction-order contention is of particular concern to **covenants**, contracts that require the spending transaction to match some pattern (e.g. returning the covenant's balance back to the same contract). Covenants typically offer a sort of "public interface", allowing multiple entities – or even the general public – to interact with a UTXO: depositing or withdrawing funds, trading tokens, casting votes, and more.

**Spend races** occur when multiple entities attempt to spend the same Bitcoin Cash UTXO. Spend races can degrade the user experience of interacting with covenants, requiring users to retry covenant transactions, and possibly preventing a user from interacting with the covenant at all.

To reduce disruptions from spend races, contracts must **carefully consider spend-race incentives**:

- To reduce frontrunning, covenants should allow actions to be submitted over time, **treating all submitted actions equally (regardless of submission time)** at some later moment.
- To disincentivize DOS attacks – e.g. where an attacker creates and rapidly broadcasts long chains of covenant transactions, spending each successive UTXO before other users can spend it in their own covenant transactions – covenants should **ensure covenant actions are authenticated and/or costly** (e.g. can only be taken once by each token holder, require some sort of fee or deposit, etc.).
- To resist censorship or orphaning of unconfirmed covenant transaction chains by malicious miners, covenants should **ensure important activity can occur over the course of many blocks**, and wallet software should maintain logic for recovering (possibly re-requesting authorization from the user) and retrying covenant transactions that are invalidated by malicious miners.

Beyond these basic strategies, this proposal enables another strategy: **multithreaded covenants** can offload logic to **"thread"** sub-covenants, allowing users to interact in parallel with multiple UTXOs. Threads aggregate state independently, allowing input or results to be "checked in" to the parent covenant in batches. Multithreaded covenants can identify authentic threads using [tracking tokens](#covenant-tracking-identity-tokens) (issued by the parent contract), and threads can authenticate both other threads and the parent covenant in the same way.

Thread design is application-specific, but valuable constructions may include:

- **Lifetimes** – to avoid situations where the parent covenant is waiting on an overactive thread (e.g. a DOS attack), threads should often have a fixed lifetime – a decrementing internal counter that prevents the thread from accepting further transactions after reaching `0`. (And leaving a check-in with the parent covenant as the only valid spending method.)
- **Heartbeats** – a derivation of lifetimes for long-lived threads, heartbeats allow a thread's lifetime to be renewed to some fixed constant after a period (by validating locktime or sequence numbers). This guarantees occasional periods of inactivity during which a check-in can be performed.
- **Proof-of-work** – some threads may have use for rate limiting by proof-of-work, requiring users to submit preimages that hash to a value using some required prefix. (Note, for most applications, fees or minimum deposits offer more uniform rate limiting.)
- modified [**zero-confirmation escrows** (ZCEs)](https://github.com/bitjson/bch-zce) – and similar miner-enforced escrows can be employed by contracts to make abusive behavior more costly.

Given typical transaction propagation speed ([99% at 2 seconds](https://github.com/bitjson/bch-zce#transaction-conflict-monitoring)), multithreaded covenant applications with reasonable spend-race disincentives can expect **minimal contention between users so long as the available thread count exceeds `2` per-interaction-per-second**. (The wallet software of real users can be expected to select evenly/randomly from available threads to maximize the likelihood of a successful transaction.) Multiple threads can be tracked by the parent covenant (e.g. using a merkle tree), and thread check-ins can be performed incrementally, so covenants can be designed to support a practically unlimited number of threads.

Finally, exceptionally active covenant applications – or applications with the potential to incentivize spend-races – should consider using **managed threads**: threads that also require the authorization of a particular key, set of keys, or non-fungible token for each submitted state change. Managed threads allow transaction submission to be ordered without contention by the entity/entities managing each thread; they can be issued either to trusted parties or via a trustless strategy, e.g. requiring a sufficiently large deposit to disincentivize frivolous thread creation.

## Multi-Covenant, Decentralized Applications

A technical demonstration has been prepared for this proposal: [Joint-Execution Decentralized Exchange (Jedex)](https://github.com/bitjson/jedex) is a multi-covenant, decentralized exchange design that supports trading between a particular fungible token and Bitcoin Cash. A full [list of demonstrated concepts](https://github.com/bitjson/jedex#demonstrated-concepts), the [application-specific token API](https://github.com/bitjson/jedex#jedex-token-api), and the application's [available transaction types](https://github.com/bitjson/jedex#transaction-flow) are each documented.
