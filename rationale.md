# Rationale

This section documents design decisions made in this specification.

## Incompatibility of Token Fungibility and Token Commitments

Advanced BCH contract use cases require strategies for transferring authenticated commitments – messages attesting to ownership, authorization, credit, debt, or other contract state – from one contract to another (a motivation behind [PMv3](https://github.com/bitjson/pmv3)). These use cases often conflicted with previous, fungibility-focused token proposals (see [Colored Coins](alternatives.md#colored-coins)).

One key insight which precipitated this proposal's bifurcated fungible/non-fungible approach is: **token fungibility and token commitments are conceptually incompatible**.

Fungible tokens are (by definition) indistinguishable from one another. Fungible token systems must allow amounts of tokens to be freely divided and re-merged without tracking the precise flow of individual token units. Conversely, nonfungible tokens (as defined by this proposal) are most useful to contracts because they offer a strategy for issuing tamper-proof messages that can be read and acted upon by other contracts.

Any token standard that attempts to combine these primitives must contend with their conceptual incompatibility – "fungible" tokens with commitments are not strictly fungible (e.g. some covenants could reject certain commitments, so wallet software must "assay" quantities of such tokens) and must have either implicit or user-defined policies for splitting and merging commitments (increasing protocol complexity and impeding standardization).

By clearly separating the fungible and non-fungible use cases, this specification is able to reduce each to a more fundamental, VM-compatible primitive. Rather than exhaustively specifying minting, transfer, or destruction "policies" at the protocol level – or creating another subsystem in which such policies are user-defined – all such policies can be specified using the existing Bitcoin Cash VM bytecode.

## Selection of Commitment Types

Commitments lift the management of data to the transaction level, allowing contracts to trust that the committed data is authentic (a commitment cannot be counterfeited). The [byte-string commitment type](readme.md#byte-string-commitments) ("non-fungible tokens") enforces only the authenticity of a particular byte string, but the more specialized [numeric commitment type](readme.md#numeric-commitments) ("fungible tokens") enables additional state management at the transaction level: the `amount` held in a numeric commitment can be freely divided among additional outputs or merged into fewer outputs. Because this behavior is enforced by the protocol, contracts can offload (to the protocol) logic that would otherwise be required to emulate this splitting/merging functionality.

Specialized commitment types could be designed for other VM data types – booleans, bitfields, hashes, signatures, and public keys, but these additional specialized commitment types have been excluded from this proposal for multiple reasons.

### Boolean Commitment Type

A specialized **boolean commitment type** offers little value because boolean values cannot be combined non-destructively: once "merged", the component boolean values cannot be extracted from a merged boolean value, so "boolean commitment merging" is equivalent to burning of the merged commitments (leaving only a single `true` or `false` commitment in their place). As such, byte-string commitments are already optimal for the representation of boolean values.

### Bitfield Commitment Type

A specialized **bitfield commitment type** could offer additional value beyond byte-string commitments because merging and recombination of bitfields is potentially non-destructive: bits set to `1` in the bitfield can be preserved within a transaction, merging any bitfields of the same category to hold `1` bits at multiple bitfield offsets within the same output. While this could reduce the size of some contracts that employ bitfields for state management, this reduction would be achieved at the cost of protocol and wallet implementation complexity:

- Transaction validation would be required to count the instances of each set bit in input bitfields and verify that outputs do not set more instances at each offset, and
- Wallet implementations would be required to carefully manage bitfields to avoid destroying set bits (by e.g. accidentally merging two bitfields that each set a bit at the same offset).

Finally, many other uses for "bitfield commitments" are not fully compatible with this simple split/merge behavior, requiring complex contract logic or byte-string commitments anyways. For example, [multithreaded covenants](examples.md#multithreaded-covenants) can represent a thread's identity using a bitfield such that [one bit is assigned to each thread](https://github.com/bitjson/jedex#demonstrated-concepts). These bitfield commitments must be carefully controlled by covenants to ensure that thread IDs cannot be moved or changed unexpectedly, so in practice, a specialized bitfield commitment type could only reduce the size of these covenants by a few bytes per transaction vs. equivalent covenants designed to use byte-string commitments. Because these slight efficiency gains apply only to a small subset of contracts, the required increase in protocol and wallet implementation complexity for a specialized bitfield commitment type cannot be justified.

### Hash Commitment Types

Two specialized **hash commitment types** could be designed to allow commitments containing hashes to be merged into a single output and later separated; the design requirements of each type depends on the secrecy of the hash preimage.

#### Secret Hash Commitment Type

Many decentralized oracle use cases require participants to commit to secret values prior to a reveal period (e.g. [sealed voting](examples.md#sealed-voting)). For a hash commitment type to be useful in these applications, the type must support merging and splitting of commitments without revealing the hashed contents.

This **secret hash commitment type** would require a protocol-standardized commitment data structure to/from which sets of hashed values can be added or removed (e.g. a standard Merkle tree), and efficient utilization within contracts would require new VM operations that allow the contract to verify that one or more hashes are included or excluded from the data structure (e.g. [`OP_MERKLEBRANCHVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0116.mediawiki)). Additionally, many use cases also require nonsecret/validated metadata (e.g. a block height, block time, a count of shares cast, etc.) in addition to the hash of the secret preimage, so a secret hash commitment type would need to either 1) support associating non-secret data with each hash, or 2) defer to byte-string commitments for such use cases.

In practice, contracts can already define equivalent, application-specific data structures and validation strategies using VM bytecode and byte-string commitments; because these efficiency gains apply only to a small subset of contracts, the required increase in protocol and wallet implementation complexity for a specialized secret hash commitment type cannot be justified.

#### Nonsecret Hash Commitment Type

A specialized **nonsecret hash commitment type** could be designed to support use cases that do not require hashed data to remain secret during merging and splitting of commitments. Users could present structured data to some contract that verifies and/or records (to its internal state) the raw data before issuing a token committing to the hash of the data; the token could be presented later to a supporting contract to demonstrate that the raw data was previously verified or recorded by the issuing contract. This nonsecret hash commitment type would support only a subset of the contracts that might be made more efficient by a [secret hash commitment type](#secret-hash-commitment-type), but this tradeoff could allow for a simpler commitment data structure (e.g. canonically-ordered, length-prefixed preimages) and simpler contract verification strategies.

As with a secret hash commitment type, efficient utilization of a nonsecret hash commitment type within contracts would require new VM operations that allow the contract to verify that one or more hashes are included or excluded from the commitment data structure (e.g. incremental hashing operations). Because byte-string commitments and the existing VM bytecode system can already be used to define equivalent, application-specific data structures and validation strategies, the required increase in protocol and wallet implementation complexity for a specialized nonsecret hash commitment type cannot be justified.

### Public Key Commitment Type

A specialized **public key commitment type** could be designed to support use cases in which a contract issues commitments containing public keys, allowing the committed public keys to be aggregated (and possibly redivided among outputs). Such a type would require the protocol-standardization of a particular public key aggregation scheme and (likely) new VM operations to fully-enable usage of the scheme within contracts.

Even with these protocol changes, a practical use case for such a commitment type has not been identified. Most contracts issuing public keys in commitments as an authorization strategy – allowing the "public key token" to be provided with a signature from the public key – could be more efficiently implemented by simply accepting the token itself as proof of authorization; providing an additional signature by the committed public key would be extraneous. In cases where the additional signature is useful, it is not clear that support for aggregating and/or redividing committed public keys would allow contracts to offload any logic to the protocol: if a contract issues commitments for multiple public keys to different entities, each entity gains nothing from aggregating their public key with other entities (effectively burning their ability to individually sign for the committed public key); likewise, if division of aggregated public keys is supported, contracts have little purpose in issuing commitments of aggregated public keys, as the token holder could always divide the aggregated public key and use the token with only one of the component public keys. As such, the required increase in protocol and wallet implementation complexity for a specialized public key commitment type cannot be justified.

### Signature Commitment Type

As with public keys, a specialized **signature commitment type** could be designed to support signature aggregation, but such a commitment type offers no additional value in contract applications. Signatures are self-authenticating data – they can be freely copied and validated directly from unlocking bytecode; any signature commitment could be replaced by an equivalent empty commitment, and the signature instead provided in the unlocking bytecode, without impacting security. As such, the required increase in protocol and wallet implementation complexity for a specialized public key commitment type cannot be justified.

## Shared Codepoint for All Tokens

Though fungible and non-fungible tokens are independent primitives, this specification defines the same `PREFIX_TOKEN` codepoint for encoding both token types.

Alternatively, additional codepoints could be reserved to save a byte for outputs that encode only tokens of one or the other type (e.g. "`PREFIX_FUNGIBLE`" and "`PREFIX_NONFUNGIBLE`"). However, this change would require matching of the additional prefixes during decoding – increasing decoding complexity in the common case (if tokens are not present on the majority of outputs) – and the potential savings are insufficient to justify expending additional codepoints.

## Behavior of Minting and Mutable Tokens

This specification includes support for adding two different "capabilities" to non-fungible tokens: `minting` and `mutable`.

Minting tokens allow the holder to create new non-fungible tokens that share the same category ID as the minting token. By implementing this minting capability for a category (rather than requiring all category tokens to be minted in a single transaction), this specification enables use cases that require both a stable category ID and long-running issuance. This is critical for both primarily-off-chain applications of non-fungible tokens (where issuers commonly require both stable identifiers and an issuer-provided commitment) and for most covenant use cases (where covenants must be able to create new commitments in the course of operation).

By implementing category-minting control as a token, minting policies can be defined using complex contracts (e.g. multisig vaults with time-based fallbacks) or even covenants (e.g. to enforce uniqueness of commitments within the category, to issue receipts or other bearer instruments to covenant users, etc.). Notably, this specification even allows minting tokens to create additional minting tokens; this is valuable for [many specialized covenants](examples.md#multithreaded-covenants) that require the ability to delegate token-creation authority to other covenants.

Mutable tokens allow the holder to create **only one new token** (i.e. "modify" the mutable token's commitment) that may again have the mutable capability.

This is a particularly critical use case for covenants, as it enables covenants to modify the commitment in a single NFT (e.g. [tracking token](examples.md#covenant-tracking-identity-tokens)) without exhaustively validating that the interaction did not unexpectedly mint new tokens (allowing the spender to impersonate the covenant). While exhaustive validation could be made efficient with new VM opcodes (like [loops](https://github.com/bitjson/bch-loops)), such validation may commonly conflict across covenants, preventing them from being used in the same transaction. As such, this proposal considers the `mutable` capability to be essential for [cross-covenant interfaces](readme.md#cross-contract-interfaces).

## Exclusion of Cloneable Capability

This proposal could specify a "cloneable" capability that would allow the spender to create new NFTs with the same commitment as the cloneable NFT. Such a capability could simplify use cases where a single commitment must be consumed by multiple contracts. However, more byte-efficient strategies are already available for implementing this behavior (e.g. ["certification tokens"](https://bitcoincashresearch.org/t/chip-2022-02-cashtokens-token-primitives-for-bitcoin-cash/725/21)), and a "cloneable" capability would both increase protocol complexity and encourage inefficient application designs that duplicate commitment data in the UTXO set.

## Avoiding Proof-of-Work for Token Data Compression

Previous token proposals require token creators to retry hashing preimages until the resulting token category ID matches required patterns. This strategy enables additional bits of information to be packed into the category ID.

In practice, such proof-of-work strategies unnecessarily complicate covenant-managed token creation. To create [ecosystems of contracts that are mutually-aware of each other contract](examples.md#covenant-tracking-identity-tokens), it is valuable to be able to predict the category ID of a token that has not yet been created; hashing strategies often preclude such planning.

Finally, at a software level, rapid retrying of hash preimages is expensive to implement and audit. Wallet software must recognize which fields may be modified, and some logic must select the parameters of each attempt. This additional flexibility presents a large surface area for exfiltration of key material (i.e. hiding parts of the private key in various modifiable structures). Signing standards for mitigating this risk may be expensive to specify and implement.

## Use of Transaction IDs as Token Category IDs

In defining token category IDs, this proposal makes a tradeoff: new token categories can only be created using outpoints with an index of `0` (UTXOs that were the 0th output of a transaction), and in exchange, **existing transaction indexes can be used to locate token genesis transactions** (token category creation transactions).

This tradeoff significantly simplifies implementations in node and indexing software, and it **allows wallet software to easily query and verify both genesis transaction information and later token history**. This can be useful for verifying token supply, inspecting data included in any genesis transaction `OP_RETURN` outputs, or performing other protocols (e.g. inspecting [Bitauth](https://github.com/bitauth/bitauth-cli) metadata).

Additionally, the one-category-per-transaction requirement is trivial to meet: token-creating wallet software can provide new category IDs using single-output (zero-confirmation) intermediate transactions; contracts that oversee token creation can require users to provide a sufficient number of category IDs (via such intermediate transactions) and need only verify that they have been provided (as the VM would reject invalid token category IDs prior to contract evaluation).

Finally, If real-world usage demonstrates demand, a future upgrade could enable token category creation with non-zero outpoint indexes by, e.g. allowing outputs with new token categories to use a hash of the full outpoint (both hash and index) in the corresponding input. (Such hashes must be 32 bytes to prevent birthday attacks.)

## One Token Prefix Per Output

Many use cases rely on the ability for one transaction output to effectively control multiple groups/types of tokens, e.g. managing multiple non-fungible tokens or trading between multiple categories of fungible tokens. This specification allows only one token prefix per output (and within it, only one NFT) for several reasons:

- **Simplified VM API** – because only one category of tokens can exist at any UTXO or output, the "token API" exposed via token inspection operations is significantly simplified: each operation requires only one index, and contracts need not handle combinatorial cases where tokens are placed at unexpected "sub-indexes" (in addition to validation across standard UTXO/output indexes). This reduces contract complexity and transaction sizes in the common case where contracts need only interact with one or two token categories.
- **Simplified data model** – limiting the data model to a single token prefix allows node implementations and indexing systems to support tokens without introducing another many-to-many relationship between some token "sub-output" and the existing output type (the most scaling-critical data type). The single-prefix limitation also allows token information to be easily denormalized into fixed-size output data structures.

Importantly, it should be noted that alternative strategies exist for effectively locking multiple (groups of) tokens to a single output. For example, a decentralized order book for trading non-fungible tokens could hold a portfolio of tokens in many separate [depository covenants](examples.md#depository-child-covenants), contracts that require the parent covenant to participate in any transactions. The sub-covenant can likewise verify the authenticity of the parent using a ["tracking" non-fungible token](examples.md#covenant-tracking-identity-tokens) that moves along with the parent covenant. (Or in cases where the sub-covenant moves with its parent in every transaction, it can also simply authenticate the parent by outpoint index after comparing with its own outpoint transaction hash.)

This parent-child strategy offers contracts practically unlimited flexibility in holding portfolios of both fungible and non-fungible tokens, and it encourages more efficient contract design across the ecosystem: rather than leaving large lists of token prefixes in a single large output (duplicating the full list in each transaction), transactions can involve only outputs with the tokens needed for a particular code path. Because contract development tooling will already require multi-output behavior, this optimization step is more likely to become widespread.

## Including Capabilities in Token Category Inspection Operations

The token category inspection operations (`OP_*TOKENCATEGORY`) in this proposal push the **concatenation of both category and capability** (if present). While token capabilities could instead be inspected with individual `OP_*TOKENCAPABILITY` operations, the behavior specified in this proposal is valuable for contract efficiency and security.

First, the combined `OP_*TOKENCATEGORY` behavior reduces contract size in a common case: every covenant that handles tokens must regularly compare both the category and capability of tokens. With the separated `OP_*TOKENCATEGORY` behavior, this common case would require at least 3 additional bytes for each occurrence – `<index> OP_*TOKENCAPABILITY OP_CAT`, and commonly, 6 or more bytes: `<index> OP_UTXOTOKENCAPABILITY OP_CAT` and `<index> OP_OUTPUTTOKENCAPABILITY OP_CAT` (`<index>` may require multiple bytes).

There are generally two other cases to consider:

- **covenants that hold mutable tokens** – these covenants are also optimized by the combined `OP_*TOKENCATEGORY` behavior. Because their mutable token can only create a single new mutable token, they need only verify that the user's transaction doesn't steal that mutable token: `OP_INPUTINDEX OP_UTXOTOKENCATEGORY <index> OP_OUTPUTTOKENCATEGORY OP_EQUALVERIFY` (saving at least 4 bytes when compared to the separated approach: `OP_INPUTINDEX OP_UTXOTOKENCATEGORY OP_INPUTINDEX OP_UTXOTOKENCAPABILITY OP_CAT <index> OP_OUTPUTTOKENCATEGORY <index> OP_OUTPUTTOKENCAPABILITY OP_CAT OP_EQUALVERIFY`).
- **covenants that hold minting tokens** – because minting tokens allow for new tokens to be minted, these covenants must exhaustively verify all outputs to ensure the user has not unexpectedly minted new tokens. (For this reason, minting tokens are likely to be held in isolated, minting-token child covenants, allowing the parent covenant to use the safer `mutable` capability.) For most outputs (verifying the output contains no tokens), both behaviors require the same bytes – `<index> OP_OUTPUTTOKENCATEGORY OP_0 OP_EQUALVERIFY`. For expected token outputs, the combined behavior requires a number of bytes similar to the separated behavior, e.g.: `<index> OP_OUTPUTTOKENCATEGORY <32> OP_SPLIT <0> OP_EQUALVERIFY <depth> OP_PICK OP_EQUALVERIFY` (combined, where `depth` holds the 32-byte category set via `OP_INPUTINDEX OP_UTXOTOKENCATEGORY <32> OP_SPLIT OP_DROP`) vs. `OP_INPUTINDEX OP_UTXOTOKENCATEGORY <index> OP_OUTPUTTOKENCATEGORY OP_EQUALVERIFY OP_INPUTINDEX OP_UTXOTOKENCAPABILITY <index> OP_OUTPUTTOKENCAPABILITY` (separated).

Beyond efficiency, this combined behavior is also **critical for the general security of the covenant ecosystem**: it makes the most secure validation (verifying both category and capability) the "default" and cheaper in terms of bytes than more lenient validation (allowing for other token capabilities, e.g. during minting).

For example, assume this proposal specified the separated behavior: if an upgradable (e.g. by shareholder vote) covenant with a tracking token is created without any other token behavior, dependent contracts may be written to check that the user has also somehow interacted with the upgradable covenant (i.e. `<index> OP_UTXOTOKENCATEGORY <expected> OP_EQUALVERIFY`). If the upgradable covenant later begins to issue tokens for any reason, a vulnerability in the dependant contract is exposed: users issued a token by the upgradable covenant can now mislead the dependent contract into believing it is being spent in a transaction with the upgradeable contract (by spending their issued token with the dependent contract). Because the dependent contract did not include a defensive `<index> OP_UTXOTOKENCAPABILITY <mutable> OP_EQUALVERIFY` (either by omission or to reduce contract size), it became vulnerable after a "public interface change". If `OP_UTXOTOKENCATEGORY` instead uses the combined behavior (as specified by this proposal) this class of vulnerabilities is eliminated.

Finally, this proposal's combined behavior preserves two codepoints in the Bitcoin Cash VM instruction set.

## Support for Zero-Length Commitments

This proposal allows for non-fungible token to have commitments with a length of `0`. This behavior matches that of stack items inside the VM, and it improves the efficiency and security of many contract constructions.

For many token categories, zero-length commitments can be used to represent concepts like memberships, authorizations, coupons, receipts, certifications, and more. In these cases, the availability of zero-length commitments often saves several bytes per transaction vs. 1-byte commitments (1 byte per token output, 1 to 2 bytes per validation) or fungible tokens (which require prior minting and handling of reserves).

Support for zero-length commitments also eliminates a possible contract vulnerability: token categories that commit to a single VM number (e.g. a counter or total in a covenant’s mutable token) could require that the next transaction perform a particular arithmetic operation on the committed number. If the required result is 0 (an empty VM stack item), and zero-length commitments were not supported, the covenant could be rendered unspendable (and funds permanently frozen). Workarounds include prefixing the committed number – costing at least 1 byte per token output (the prefix) and 2-4 bytes per validation (`<prefix> OP_CAT`, `<1> OP_SPLIT`) – or padding the committed number – costing 2-7 bytes per token output (`<8> OP_NUM2BIN`), and 1 byte per validation (`OP_BIN2NUM`). Supporting zero-length commitments avoids the need for contract audits to identify this issue or increase transaction sizes with workarounds.

Notably, zero-length commitments introduce a potential ambiguity in the state exposed to contracts by `OP_UTXOTOKENCOMMITMENT` and `OP_OUTPUTTOKENCOMMITMENT`; these operations return VM Number `0` for outputs with either 1) no NFT or 2) an immutable NFT with a commitment of length `0`. However, this ambiguity does not impact contract security, and it rarely increases transaction sizes. Contracts can differentiate between the cases by requiring the output in question to contain no fungible tokens (`OP_*TOKENAMOUNT <0> OP_EQUALVERIFY`), and wallets that are designed to interact with such contracts can split UTXOs containing both fungible tokens and zero-length-commitment, immutable NFTs into separate UTXOs prior to performing actions that require a zero-length-commitment, immutable NFT from a token category for which fungible tokens also exist.

Alternatively, this ambiguity could also be addressed by modifying `OP_UTXOTOKENCATEGORY` and `OP_OUTPUTTOKENCATEGORY` to include a final byte for "no capability" (i.e. an immutable NFT) rather than returning only the 32-byte token category. However, such a change would [de-optimize many common cases](#including-capabilities-in-token-category-inspection-operations) – requiring 3 additional bytes for most token validation – while only improving the efficiency of a rare case (validation of zero-length, immutable NFTs from categories that include fungible tokens) for which many efficient alternatives exist. Even for contract systems in which the rare case is applicable, this would increase transaction sizes (saving 2-3 bytes for the rare case, but costing 3 bytes per category validation in the surrounding contract).

## Limitation of Non-Fungible Token Commitment Length

This specification limits the length of non-fungible token commitments to `40` bytes; this restrains excess growth of the UTXO set and reduces the resource requirements of typical wallets and indexers.

By committing to a hash, contracts can commit to an unlimited collection of data (e.g. using a merkle tree). For resistance to [birthday attacks](https://bitcoincashresearch.org/t/p2sh32-a-long-term-solution-for-80-bit-p2sh-collision-attacks/750), covenants should typically use `32` byte hashes. This proposal expands this minimum requirement by `8` bytes to include an additional (padded, maximum-length) VM number. This is particularly valuable for covenants that are optimized to use multiple types of commitment structures and must concisely indicate their current internal "mode" to other contracts (e.g. re-organizing an unbalanced merkle tree of contract state for efficiency when the covenant enters "voting" mode). This additional 8 bytes also provides adequate space for higher-level standards to specify prefixes for committed hashes (e.g. marking identifiers to content-addressable storage).

## Inclusion of Token-Aware CashAddresses

This proposal adds two new types to the existing CashAddress format: `Token-Aware P2PKH` (`2`) and `Token-Aware P2SH` (`3`). By clearly designating "token-aware" equivalents of the existing address types, users are protected from unknowingly sending tokens to addresses where they may be inaccessible to the intended recipient.

By design, token support in most software remains optional, and many simple wallets may never have the need to implement token support (eliminating the need for token management user interfaces, simplifying UTXO management, and reducing maintenance costs). By introducing new, opt-in, token-aware CashAddress types, token-unaware software can continue to signal its inability to receive tokens, avoiding [lost funds and poor user experiences](https://github.com/bitjson/cashtokens/issues/32#issuecomment-1188122202).

## Recommendation of `SIGHASH_UTXOS` for Multi-Entity Transactions

This proposal recommends that all wallets enable the `SIGHASH_UTXOS` signing serialization flag when creating signatures for multi-entity transactions, including both 1) transactions where signatures are collected from multiple keys and assembled into a single transaction, and 2) transactions involving contracts that can be influenced by multiple entities (e.g. covenants). This recommendation eliminates several classes of vulnerabilities affecting light wallets and improves the privacy of all users of multi-entity transactions.

`SIGHASH_UTXOS` eliminates a set of potential vulnerabilities in light wallets and offline signing devices including [unexpected fee burning](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-August/014843.html), [unexpected token burning, theft, or unexpected actions in decentralized applications](https://github.com/bitjson/cashtokens/issues/22#issuecomment-1179333487).

Alternatively, signers could download and verify the spent output of each transaction referenced by each input prior to signing vulnerable transactions, but this approach would require significantly higher bandwidth, memory, and/or storage requirements for many light wallet and offline signing devices, impeding support for interacting with tokens or decentralized applications.

While these potential vulnerabilities do not impact signers with a full view of the necessary data (e.g. wallets with integrated, fully-validating nodes), because a subset of light wallets must enable `SIGHASH_UTXOS` for security, enabling `SIGHASH_UTXOS` for all signers improves privacy by maximizing the anonymity set of the resulting transactions. As such, this proposal recommends use of `SIGHASH_UTXOS` for all multi-entity transactions, even for non-vulnerable wallets.

## Limitation of Fungible Token Supply

Token validation has the effect of limiting fungible token supply to the maximum VM number (`9223372036854775807`). This is important for both contract usage and the token application ecosystem.

Consider decentralized exchange covenants that support submission of new liquidity pools: if it is possible to create a fungible token output with an amount greater than the maximum VM number, there is a class of vulnerabilities by which 1) the decentralized exchange fails to validate total amounts of newly submitted assets, 2) the error-producing value becomes embedded in the the covenant, and 3) the amount invalidates important spending paths, i.e. a denial of service (possibly permanently locking all funds). Because practically all contracts that handle arbitrary fungible tokens would need to employ this type of validation, limiting amounts to `9223372036854775807` reduces the size and complexity of a wide range of contracts.

Across the wider token application ecosystem (node software, indexers, and wallet software), the limit also makes maximum token supply simpler to calculate and display in user interfaces. Arbitrary-precision arithmetic is not necessary, and total supply has a maximum digit length of `19`.

Additionally, while a cap of `9223372036854775807` is likely sufficient for most cases (e.g. supporting $92 quadrillion at a precision of 1 cent), it is trivial to design contracts that consider two different token categories to be interchangeable. So while each token category is limited, a single genesis transaction can create a practically unlimited number of token categories, each with the maximum supply. Contracts can be designed to consider any of a set of token categories to be completely fungible, allowing practically unlimited "virtual" token amounts (but without requiring the wider token ecosystem to perform arbitrary-precision arithmetic or display the larger amounts in generalized user interfaces).

Finally, the Bitcoin Cash currency itself is significantly less granular than this limit – the maximum satoshi value is less than `2^53`, while fungible token supply extends to `2^63`. An upgrade providing for greater divisibility of fungible tokens should likely occur as part of an upgrade that increases the divisibility of the Bitcoin Cash currency itself (A.K.A. a "fractional satoshis" upgrade).

## Specification of Token Supply Definitions

The supply of covenant-issued tokens will often be inherently analyzable using the [supply-calculation algorithms](readme.md#fungible-token-supply-definitions) included in this specification. For example, all covenants that use a [tracking token](examples.md#covenant-tracking-identity-tokens) and retain a supply of unissued tokens will have an easily-calculable [circulating supply](readme.md#circulating-supply) (without requiring wallets/indexers to understand the implementation details of the contract).

By explicitly including these supply-calculation algorithms in this specification, token issuers are more likely to be aware of this reality, choosing to issue tokens via a strategy that conforms to common user expectations (improving data availability and compatibility across the ecosystem).
