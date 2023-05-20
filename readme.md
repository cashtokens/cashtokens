# CHIP-2022-02-CashTokens: Token Primitives for Bitcoin Cash

        Title: Token Primitives for Bitcoin Cash
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Final
        Initial Publication Date: 2022-02-22
        Final Revision Date: 2023-5-20
        Version: 2.2.2

<details>

<summary><strong>Table of Contents</strong></summary>

- [Summary](#summary)
- [Deployment](#deployment)
- [Motivation](#motivation)
- [Benefits](#benefits)
- [Technical Summary](#technical-summary)
- [Technical Specification](#technical-specification)
- [Usage Examples](#usage-examples)
- [Rationale](#rationale)
- [Prior Art & Alternatives](#prior-art--alternatives)
- [Stakeholder Responses & Statements](#stakeholder-responses--statements)
- [Test Vectors](#test-vectors)
- [Implementations](#implementations)
- [Feedback & Reviews](#feedback--reviews)
- [Acknowledgements](#acknowledgements)
- [Changelog](#changelog)
- [Copyright](#copyright)

</details>

## Summary

This proposal enables two new primitives on Bitcoin Cash: **fungible tokens** and **non-fungible tokens**.

### Terms

A **token** is an asset – distinct from the Bitcoin Cash currency – that can be created and transferred on the Bitcoin Cash network.

**Non-Fungible tokens (NFTs)** are a token type in which individual units cannot be merged or divided – each NFT contains a **commitment**, a short byte string attested to by the issuer of the NFT.

**Fungible tokens** are a token type in which individual units are undifferentiated – groups of fungible tokens can be freely divided and merged without tracking the identity of individual tokens (much like the Bitcoin Cash currency).

## Deployment

Deployment of this specification is proposed for the May 2023 upgrade.

- Activation is proposed for `1668513600` [MTP](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki), (`2022-11-15T12:00:00.000Z`) on [`chipnet`](https://bitcoincashresearch.org/t/staging-chips-on-testnet/573/10).
- Activation is proposed for `1684152000` [MTP](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki), (`2023-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, `testnet4`, and `scalenet`.

## Motivation

Bitcoin Cash contracts lack primitives for issuing messages that can be verified by other contracts, preventing the development of decentralized application ecosystems on Bitcoin Cash.

### Contract-Issued Commitments

In the context of the Bitcoin Cash virtual machine (VM), a **commitment** can be defined as an irrevocable message that was provably issued by a particular entity. Two forms of commitments are currently available to Bitcoin Cash contracts:

- **Transaction signatures** – a commitment made by a private key attesting to the signing serialization of a transaction.
- **Data signatures** – a commitment made by a private key attesting to the hash of an arbitrary message (introduced in 2018 by [`OP_CHECKDATASIG`](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/op_checkdatasig.md)).

Each of these commitment types require the presence of a _trusted private key_. Because contracts cannot themselves hold a private key, any use case that requires a contract to issue a verifiable message must necessarily rely on trusted entities to provide signatures<sup>1</sup>. This limitation prevents Bitcoin Cash contracts from offering or using **decentralized oracles** – multiparty consensus systems that produce verifiable messages upon which other contracts can act.

By providing a commitment primitive that can be used directly by contracts, the Bitcoin Cash contract system can support advanced, decentralized applications without increasing transaction or block validation costs.

<details><summary>Notes</summary>

1. Signature aggregation schemes can enable contracts to issue some commitments with reduced trust (e.g. by requiring a quorum of always-online entities to join a multiparty signing process), but these schemes typically require active coordination, carefully-designed participation incentives, fallback strategies, and other significant fixed costs (e.g. always-online servers). In practice, few such systems are able to maintain sufficient traction to continue functioning, and notably, forcing a contract to rely on these systems is arbitrary and wasteful (in terms of network bandwidth and validation costs) when their purpose is simply to attest to a result already produced by that contract.

</details>

#### Byte-String Commitments

The most general type of contract-issued commitment is a simple string of bytes. This type can be used to commit to any type of contract state: certifications of ownership, authorizations, credit, debt, contract-internal time or epoch, vote counts, receipts (e.g. to support refunds or future redemption), etc. Identities may commit to state within a hash structure (e.g. a merkle tree), or – to reduce transaction sizes – as a raw byte string (e.g. public keys, static numbers, boolean values).

In this proposal, byte-string commitments are called **non-fungible tokens**.

#### Numeric Commitments

**The Bitcoin Cash virtual machine (VM) supports two primary data types in VM bytecode evaluation: byte strings and numbers**. Given the existence of a primitive allowing contracts to commit to byte strings, another commitment primitive can be inferred: **numeric commitments**.

Numeric commitments are a specialization of byte-string commitments – they are commitments with numeric values which can be divided and merged in the same way as the Bitcoin Cash currency. With numeric commitments, contracts can efficiently represent fractional parts of abstract concepts – shares, pegged assets, bonds, loans, options, tickets, loyalty points, voting outcomes, etc.

While many use cases for numeric commitments can be emulated with only byte-string commitments, a numeric primitive enables many contracts to reduce or offload state management altogether (e.g. [shareholder voting](examples.md#voting-with-fungible-tokens)), simplifying contract audits and reducing transaction sizes.

In this proposal, numeric commitments are called **fungible tokens**.

## Benefits

By enabling token primitives on Bitcoin Cash, this proposal offers several benefits.

### Cross-Contract Interfaces

Using **non-fungible tokens** (NFTs), contracts can **create messages that can be read by other contracts**. These messages are impersonation-proof: other contracts can safely read and act on the commitment, certain that it was produced by the claimed contract.

With contract interoperability, behavior can be broken into clusters of smaller, coordinating contracts, **reducing transaction sizes**. This interoperability further enables [covenants](examples.md#usage-examples) to communicate over **public interfaces**, allowing diverse ecosystems of compatible covenants to work together, even when developed and deployed separately.

Critically, this cross-contract interaction can be achieved within the "stateless" transaction model employed by Bitcoin Cash, rather than coordinating via shared global state. This allows Bitcoin Cash to support comparable contract functionality while retaining its [>1000x efficiency advantage](https://blog.bitjson.com/pmv3-build-decentralized-applications-on-bitcoin-cash/#stateless-network-stateful-covenants) in transaction and block validation.

### Decentralized Applications

Beyond enabling covenants to interoperate with other covenants, these token primitives allow for byte-efficient representations of complex internal state – supporting [advanced, decentralized applications](https://github.com/bitjson/jedex) on Bitcoin Cash.

**Non-fungible tokens** are critical for coordinating activity trustlessly between multiple covenants, enabling [covenant-tracking tokens](examples.md#covenant-tracking-identity-tokens), [depository child covenants](examples.md#depository-child-covenants), [multithreaded covenants](examples.md#multithreaded-covenants), and other constructions in which a particular covenant instance must be authenticated.

**Fungible tokens** are valuable for covenants to represent on-chain assets – e.g. voting shares, utility tokens, collateralized loans, prediction market options, etc. – and implement [complex coordination tasks](examples.md#voting-with-fungible-tokens) – e.g. liquidity-pooling, auctions, voting, sidechain withdrawals, spin-offs, mergers, and more.

### Universal Token Primitives

By exposing basic, consensus-validated token primitives, this proposal supports the development of higher-level, interoperable token standards (e.g. [SLP](https://slp.dev/)). Token primitives can be held by any contract, wallets can easily verify the authenticity of a token or group of tokens, and tokens cannot be inadvertently destroyed by wallet software that does not support tokens.

## Technical Summary

1. A token **category** can include both non-fungible and fungible tokens, and every category is represented by a 32-byte category identifier – the transaction ID of the outpoint spent to create the category.
   1. All fungible tokens for a category must be created when the token category is created, ensuring the total supply within a category remains below the maximum VM number.
   2. Non-fungible tokens may be created either at category creation or in later transactions that spend tokens with `minting` or `mutable` capabilities for that category.
2. Transaction outputs are extended to support four new `token` fields; every output can include one **non-fungible token** and any amount of **fungible tokens** from a single token category.
3. [Token inspection opcodes](#token-inspection-operations) allow contracts to operate on tokens, enabling [cross-contract interfaces and decentralized applications](examples.md#usage-examples).

### Transaction Output Data Model

This proposal extends the data model of transaction outputs to add four new `token` fields: token `category`, non-fungible token `capability`, non-fungible token `commitment`, and fungible token `amount`.

| Existing Fields  | Description                                                                                                                                                    |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Value            | The value of the output in satoshis, the smallest unit of bitcoin cash. (A.K.A. `vout`)                                                                        |
| Locking Bytecode | The VM bytecode used to encumber this transaction output. (A.K.A. `scriptPubKey`)                                                                              |
| **Token Fields** | (New, optional fields added by this proposal:)                                                                                                                 |
| Category ID      | The 32-byte ID of the token category to which the token(s) in this output belong. This field is omitted if no tokens are present.                              |
| Capability       | The capability of the NFT held in this output: `none`, `mutable`, or `minting`. This field is omitted if no NFT is present.                                    |
| Commitment       | The commitment contents of the NFT held in this output (`0` to `40` bytes). This field is omitted if no NFT is present.                                        |
| Amount           | The number of fungible tokens held in this output (an integer between `1` and `9223372036854775807`). This field is omitted if no fungible tokens are present. |

<details>

<summary><strong>Transaction Output JSON Format</strong></summary>

The following snippet demonstrates a JSON representation of a transaction output using TypeScript types.

A new, optional `token` property is added, and the existing `lockingBytecode` and `valueSatoshis` properties are unmodified.

Note, this type is used by the [test vectors](#test-vectors).

```ts
/**
 * Data type representing a Transaction Output.
 */
type Output = {
  /**
   * The bytecode used to encumber this transaction output. To spend the output,
   * unlocking bytecode must be included in a transaction input that – when
   * evaluated before the locking bytecode – completes in a valid state.
   *
   * A.K.A. `scriptPubKey` or "locking script"
   */
  lockingBytecode: Uint8Array;

  /**
   * The CashToken contents of this output. This property is only defined if the
   * output contains one or more tokens.
   */
  token?: {
    /**
     * The number of fungible tokens held in this output.
     *
     * Because `Number.MAX_SAFE_INTEGER` (`9007199254740991`) is less than the
     * maximum token amount (`9223372036854775807`), this value is encoded as
     * a `bigint`. (Note, because standard JSON does not support `bigint`, this
     * value must be converted to and from a `string` to pass over the network.)
     */
    amount: bigint;
    /**
     * The 32-byte token category ID to which the token(s) in this output belong
     * in big-endian byte order. This is the byte order typically seen in block
     * explorers and user interfaces (as opposed to little-endian byte order,
     * which is used in standard P2P network messages).
     */
    category: Uint8Array;
    /**
     * If present, the non-fungible token (NFT) held by this output. If the
     * output does not include a non-fungible token, `undefined`.
     */
    nft?: {
      /**
       * The capability of this non-fungible token.
       */
      capability: 'none' | 'mutable' | 'minting';

      /**
       * The commitment contents included in the non-fungible token held in
       * this output.
       */
      commitment: Uint8Array;
    };
  };

  /**
   * The value of the output in satoshis, the smallest unit of bitcoin cash.
   */
  valueSatoshis: number;
};
```

</details>

## Technical Specification

<details>

<summary><strong>Subsections</strong></summary>

- [Token Categories](#token-categories)
- [Token Types](#token-types)
- [Token Behavior](#token-behavior)
  - [Universal Token Behavior](#universal-token-behavior)
  - [Non-Fungible Token Behavior](#non-fungible-token-behavior)
  - [Fungible Token Behavior](#fungible-token-behavior)
- [Token Encoding](#token-encoding)
  - [Token Prefix](#token-prefix)
  - [Token Prefix Validation](#token-prefix-validation)
  - [Token Prefix Standardness](#token-prefix-standardness)
- [Token Encoding Activation](#token-encoding-activation)
- [Token-Aware Transaction Validation](#token-aware-transaction-validation)
  - [Token Validation Algorithm](#token-validation-algorithm)
- [Token Inspection Operations](#token-inspection-operations)
  - [Existing Introspection Operations](#existing-introspection-operations)
  - [Interpretation of Signature Preimage Inspection](#interpretation-of-signature-preimage-inspection)
- [Signing Serialization of Tokens](#signing-serialization-of-tokens)
- [`SIGHASH_UTXOS`](#sighash_utxos)
- [Double Spend Proof Support](#double-spend-proof-support)
- [CashAddress Token Support](#cashaddress-token-support)
- [Token-Aware BIP69 Sorting Algorithm](#token-aware-bip69-sorting-algorithm)
- [Fungible Token Supply Definitions](#fungible-token-supply-definitions)
  - [Genesis Supply](#genesis-supply)
  - [Total Supply](#total-supply)
  - [Reserved Supply](#reserved-supply)
  - [Circulating Supply](#circulating-supply)

</details>

Token primitives are defined, token encoding and activation are specified, and six new token inspection opcodes are introduced. Transaction validation and transaction signing serialization is modified to support tokens, `SIGHASH_UTXOS` is specified, BIP69 sorting is extended to support tokens, and CashAddress `types` with token support are defined.

### Token Categories

Every token belongs to a **token category** specified via an immutable, 32-byte **Token Category ID** assigned in the category's **genesis transaction** – the transaction in which the token category is initially created.

Every token category ID is a transaction ID: the ID must be selected from the inputs of its genesis transaction, and only **token genesis inputs** – inputs which spend output `0` of their parent transaction – are eligible (i.e. outpoint transaction hashes of inputs with an outpoint index of `0`). As such, implementations can locate the genesis transaction of any category by identifying the transaction that spent the `0`th output of the transaction referenced by the category ID. (See [Use of Transaction IDs as Token Category IDs](rationale.md#use-of-transaction-ids-as-token-category-ids).)

Note that because every transaction has at least one output, every transaction ID can later become a token category ID.

![CashToken Creation](./figures/cashtoken-creation.svg)
_Figure 1. Two new token categories are created by transaction `c3a601...`. The first category (<span style="color:#8a11b2">■</span>`b201a0...`) is created by spending the 0th output of transaction `b201a0...`; for this category, a supply of `150` fungible tokens are created across two outputs (`100` and `50`). The second category (<span style="color:#3ab211">▲</span>`a1efcd...`) is created by spending the 0th output of transaction `a1efcd...`; for this category, a supply of `100` fungible tokens are created across two outputs (`20` and `80`), and two NFTs are created (each with a commitment of `0x010203`)._

### Token Types

Two token types are introduced: **fungible tokens** and **non-fungible tokens**. Fungible tokens have only one property: a 32-byte `category`. Non-fungible tokens have three properties: a 32-byte `category`, a `0` to `40` byte `commitment`, and a `capability` of `minting`, `mutable`, or `none`.

### Token Behavior

Token behavior is enforced by the [**token validation algorithm**](#token-validation-algorithm). This algorithm has the following effects:

#### Universal Token Behavior

1.  A single transaction can create multiple new token categories, and each category can contain both fungible and non-fungible tokens.
2.  Tokens can be implicitly destroyed by omission from a transaction's outputs.
3.  Each transaction output can contain zero or one non-fungible token and any `amount` of fungible tokens, but all tokens in an output must share the same token category.

#### Non-Fungible Token Behavior

1.  A transaction output can contain zero or one **non-fungible token**.
2.  Non-fungible tokens (NFTs) of a particular category are created either in the category's genesis transaction or by later transactions that spend `minting` or `mutable` tokens of the same category.
3.  It is possible for multiple NFTs of the same category to carry the same commitment. (Though uniqueness can be enforced by covenants.)
4.  **Minting tokens** (NFTs with the `minting` capability) allow the spending transaction to create any number of new NFTs of the same category, each with any commitment and (optionally) the `minting` or `mutable` capability.
5.  Each **Mutable token** (NFTs with the `mutable` capability) allows the spending transaction to create one NFT of the same category, with any commitment and (optionally) the `mutable` capability.
6.  **Immutable tokens** (NFTs without a capability) cannot have their commitment modified when spent.

#### Fungible Token Behavior

1.  A transaction output can contain any `amount` of fungible tokens from a single category.
2.  All fungible tokens of a category are created in that category's genesis transaction; their combined `amount` may not exceed `9223372036854775807`.
3.  A transaction can spend fungible tokens from any number of UTXOs to any number of outputs, so long as the sum of output `amount`s do not exceed the sum of input `amount`s (for each token category).

Note that fungible tokens behave independently from non-fungible tokens: non-fungible tokens are never counted in the `amount`, and the existence of `minting` or `mutable` NFTs in a transaction's inputs do not allow for new fungible tokens to be created.

### Token Encoding

Tokens are encoded in outputs using a **token prefix**, a data structure that can encode a token category, zero or one non-fungible token (NFT), and an amount of fungible tokens (FTs).

For backwards-compatibility with existing transaction decoding implementations, a transaction output's token prefix (if present) is encoded before index `0` of its locking bytecode, and the `CompactSize` length preceding the two fields is increased to cover both fields (such that the length could be renamed `token_prefix_and_locking_bytecode_length`). The token prefix is not part of the locking bytecode and must not be included in bytecode evaluation. To illustrate, after deployment, the serialized output format becomes:

```
<satoshi_value> <token_prefix_and_locking_bytecode_length> [PREFIX_TOKEN <token_data>] <locking_bytecode>
```

#### Token Prefix

`PREFIX_TOKEN` is defined at codepoint `0xef` (`239`) and indicates the presence of a token prefix:

```
PREFIX_TOKEN <category_id> <token_bitfield> [nft_commitment_length nft_commitment] [ft_amount]
```

1. `<category_id>` – After the `PREFIX_TOKEN` byte, a 32-byte **Token Category ID** is required, encoded in `OP_HASH256` byte order<sup>1</sup>.
2. `<token_bitfield>` - A bitfield encoding two 4-bit fields is required:
   1. `prefix_structure` (`token_bitfield & 0xf0`) - 4 bitflags, defined at the higher half of the bitfield, indicating the structure of the token prefix:
      1. `0x80` (`0b10000000`) - `RESERVED_BIT`, must be unset.
      2. `0x40` (`0b01000000`) - `HAS_COMMITMENT_LENGTH`, the prefix encodes a commitment length and commitment.
      3. `0x20` (`0b00100000`) - `HAS_NFT`, the prefix encodes a non-fungible token.
      4. `0x10` (`0b00010000`) - `HAS_AMOUNT`, the prefix encodes an amount of fungible tokens.
   2. `nft_capability` (`token_bitfield & 0x0f`) – A 4-bit value, defined at the lower half of the bitfield, indicating the non-fungible token capability, if present.
      1. If not `HAS_NFT`: must be `0x00`.
      2. If `HAS_NFT`:
         1. `0x00` – No capability – the encoded non-fungible token is an **immutable token**.
         2. `0x01` – The **`mutable` capability** – the encoded non-fungible token is a **mutable token**.
         3. `0x02` – The **`minting` capability** – the encoded non-fungible token is a **minting token**.
         4. Values greater than `0x02` are reserved and must not be used.
3. If `HAS_COMMITMENT_LENGTH`:
   1. `commitment_length` – A **commitment length** is required (minimally-encoded in `CompactSize` format<sup>2</sup>) with a minimum value of `1` (`0x01`).
   2. `commitment` – The non-fungible token's **commitment** byte string of `commitment_length` is required.
4. If `HAS_AMOUNT`:
   1. `ft_amount` – An amount of **fungible tokens** is required (minimally-encoded in `CompactSize` format<sup>2</sup>) with a minimum value of `1` (`0x01`) and a maximum value equal to the maximum VM number, `9223372036854775807` (`0xffffffffffffff7f`).

<details>

<summary>Notes</summary>

1. This is the byte order produced/required by all BCH VM operations which employ SHA-256 (including `OP_SHA256` and `OP_HASH256`), the byte order used for outpoint transaction hashes in the P2P transaction format, and the byte order produced by most SHA-256 libraries. For reference, the genesis block header in this byte order is little-endian – `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000` – and can be produced by this script: `<0x0100000000000000000000000000000000000000000000000000000000000000000000003ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4a29ab5f49ffff001d1dac2b7c> OP_HASH256`. (Note, this is the opposite byte order as is commonly used in user interfaces like block explorers.)
2. The **`CompactSize` Format** is a variable-length, little-endian, positive integer format used to indicate the length of the following byte array in Bitcoin Cash P2P protocol message formats (present since the protocol's publication in 2008). The format historically allowed some values to be encoded in multiple ways; token prefixes must always use minimally-encoded/canonically-encoded `CompactSize`s, e.g. the value `1` must be encoded as `0x01` rather than `0xfd0100`, `0xfe0100000`, or `0xff010000000000000`.

</details>

#### Token Prefix Validation

1. By consensus, `commitment_length` is limited to `40` (`0x28`), but future upgrades may increase this limit. Implementers are advised to ensure that values between `253` (`0xfdfd00`) and `65535` (`0xfdffff`) can be parsed. (See [Non-Fungible Token Commitment Length](rationale.md#non-fungible-token-commitment-length).)
2. A token prefix encoding no tokens (both `HAS_NFT` and `HAS_AMOUNT` are unset) is invalid.
3. A token prefix encoding `HAS_COMMITMENT_LENGTH` without `HAS_NFT` is invalid.
4. A token prefix where `HAS_NFT` is unset must encode `nft_capability` of `0x00`.

#### Token Prefix Standardness

Implementations must recognize otherwise-standard outputs with token prefixes as **standard**.

<details>

<summary><strong>Token Prefix Encoding Test Vectors</strong></summary>

The following test vectors demonstrate valid, reserved, and invalid token prefix encodings. The token category ID is `0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb` and commitments use repetitions of `0xcc`.

For the complete set of test vectors, see [Test Vectors](#test-vectors).

#### Valid Token Prefix Encodings

| Description                                        | Encoded (Hex)                                                                                                                                                              |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| no NFT; 1 fungible                                 | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb1001`                                                                                                   |
| no NFT; 252 fungible                               | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb10fc`                                                                                                   |
| no NFT; 253 fungible                               | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb10fdfd00`                                                                                               |
| no NFT; 9223372036854775807 fungible               | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb10ffffffffffffffff7f`                                                                                   |
| 0-byte immutable NFT; 0 fungible                   | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb20`                                                                                                     |
| 0-byte immutable NFT; 1 fungible                   | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb3001`                                                                                                   |
| 0-byte immutable NFT; 253 fungible                 | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb30fdfd00`                                                                                               |
| 0-byte immutable NFT; 9223372036854775807 fungible | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb30ffffffffffffffff7f`                                                                                   |
| 1-byte immutable NFT; 0 fungible                   | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb6001cc`                                                                                                 |
| 1-byte immutable NFT; 252 fungible                 | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7001ccfc`                                                                                               |
| 2-byte immutable NFT; 253 fungible                 | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7002ccccfdfd00`                                                                                         |
| 10-byte immutable NFT; 65535 fungible              | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb700accccccccccccccccccccfdffff`                                                                         |
| 40-byte immutable NFT; 65536 fungible              | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7028ccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccfe00000100`         |
| 0-byte, mutable NFT; 0 fungible                    | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb21`                                                                                                     |
| 0-byte, mutable NFT; 4294967295 fungible           | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb31feffffffff`                                                                                           |
| 1-byte, mutable NFT; 0 fungible                    | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb6101cc`                                                                                                 |
| 1-byte, mutable NFT; 4294967296 fungible           | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7101ccff0000000001000000`                                                                               |
| 2-byte, mutable NFT; 9223372036854775807 fungible  | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7102ccccffffffffffffffff7f`                                                                             |
| 10-byte, mutable NFT; 1 fungible                   | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb710acccccccccccccccccccc01`                                                                             |
| 40-byte, mutable NFT; 252 fungible                 | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7128ccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccfc`                 |
| 0-byte, minting NFT; 0 fungible                    | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb22`                                                                                                     |
| 0-byte, minting NFT; 253 fungible                  | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb32fdfd00`                                                                                               |
| 1-byte, minting NFT; 0 fungible                    | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb6201cc`                                                                                                 |
| 1-byte, minting NFT; 65535 fungible                | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7201ccfdffff`                                                                                           |
| 2-byte, minting NFT; 65536 fungible                | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7202ccccfe00000100`                                                                                     |
| 10-byte, minting NFT; 4294967297 fungible          | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb720accccccccccccccccccccff0100000001000000`                                                             |
| 40-byte, minting NFT; 9223372036854775807 fungible | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7228ccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccffffffffffffffff7f` |

##### Reserved Token Prefix Encodings

These encodings are valid but disabled due to excessive `commitment_length`s. Transactions attempting to create outputs with these token prefixes are currently rejected by consensus, but future upgrades may increase the maximum valid `commitment_length`.

| Description                                        | Encoded (Hex)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 41-byte immutable NFT; 65536 fungible              | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7029ccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccfe00000100`                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| 41-byte, mutable NFT; 252 fungible                 | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7129ccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccfc`                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| 41-byte, minting NFT; 9223372036854775807 fungible | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7229ccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccffffffffffffffff7f`                                                                                                                                                                                                                                                                                                                                                                                                                           |
| 253-byte, immutable NFT; 0 fungible                | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb60fdfd00cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` |

#### Invalid Token Prefix Encodings

| Reason                                                                                                 | Encoded (Hex)                                                                              |
| ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| Token prefix must encode at least one token                                                            | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb00`                     |
| Token prefix must encode at least one token (0 fungible)                                               | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb1000`                   |
| Token prefix requires a token category ID                                                              | `ef`                                                                                       |
| Token category IDs must be 32 bytes                                                                    | `efbbbbbbbb1001`                                                                           |
| Missing token bitfield                                                                                 | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`                       |
| Token bitfield sets reserved bit                                                                       | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb9001`                   |
| Unknown capability (0-byte NFT, capability 3)                                                          | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb23`                     |
| Has commitment length without NFT (1 fungible)                                                         | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb5001`                   |
| Prefix encodes a capability without an NFT                                                             | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb1101`                   |
| Commitment length must be specified (immutable token)                                                  | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb60`                     |
| Commitment length must be specified (mutable token)                                                    | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb61`                     |
| Commitment length must be specified (minting token)                                                    | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb62`                     |
| Commitment length must be minimally-encoded                                                            | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb60fd0100cc`             |
| If specified, commitment length must be greater than 0                                                 | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb6000`                   |
| Not enough bytes remaining in locking bytecode to satisfy commitment length (0/1 bytes)                | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb6001`                   |
| Not enough bytes remaining in locking bytecode to satisfy commitment length (mutable token, 0/1 bytes) | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb6101`                   |
| Not enough bytes remaining in locking bytecode to satisfy commitment length (mutable token, 1/2 bytes) | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb6102cc`                 |
| Not enough bytes remaining in locking bytecode to satisfy commitment length (minting token, 1/2 bytes) | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb6202cc`                 |
| Not enough bytes remaining in locking bytecode to satisfy token amount (no NFT, 1-byte amount)         | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb10`                     |
| Not enough bytes remaining in locking bytecode to satisfy token amount (no NFT, 2-byte amount)         | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb10fd00`                 |
| Not enough bytes remaining in locking bytecode to satisfy token amount (no NFT, 4-byte amount)         | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb10fe000000`             |
| Not enough bytes remaining in locking bytecode to satisfy token amount (no NFT, 8-byte amount)         | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb10ff00000000000000`     |
| Not enough bytes remaining in locking bytecode to satisfy token amount (immutable NFT, 1-byte amount)  | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7001cc`                 |
| Not enough bytes remaining in locking bytecode to satisfy token amount (immutable NFT, 2-byte amount)  | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7001ccfd00`             |
| Not enough bytes remaining in locking bytecode to satisfy token amount (immutable NFT, 4-byte amount)  | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7001ccfe000000`         |
| Not enough bytes remaining in locking bytecode to satisfy token amount (immutable NFT, 8-byte amount)  | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb7001ccff00000000000000` |
| Token amount must be specified                                                                         | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb30`                     |
| If specified, token amount must be greater than 0 (no NFT)                                             | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb1000`                   |
| If specified, token amount must be greater than 0 (0-byte NFT)                                         | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb3000`                   |
| Token amount must be minimally-encoded                                                                 | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb10fd0100`               |
| Token amount (9223372036854775808) may not exceed 9223372036854775807                                  | `efbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb30ff0000000000000080`   |

</details>

### Token Encoding Activation

**Pre-activation token-forgery outputs (PATFOs)** are transaction outputs mined in blocks prior to activation of this specification where locking bytecode index `0` is set to the `PREFIX_TOKEN` codepoint.

Prior to activation, PATFOs remain **nonstandard** but do not invalidate the transaction by consensus. Because they can still be mined in valid blocks, PATFOs can be used to prepare outputs that, after activation of this specification, could encode tokens for which [Token-Aware Transaction Validation](#token-aware-transaction-validation) was not enforced (producing token categories that do not map to a confirmed transaction hash or have a fungible token supply exceeding the maximum amount).

Note, even properly-encoded token outputs included in transactions mined prior to activation are considered PATFOs, regardless of whether or not the transaction would pass token-aware transaction validation after activation. Due to the possibility of a chain re-organization impacting the precise activation time, token issuers are advised to wait until activation is confirmed to a depth of at least 11 blocks before broadcasting critical transactions involving tokens.

PATFOs are provably unspendable<sup>1</sup>; all software implementing this specification should immediately mark PATFOs as unspendable and/or logically treat them as such for the purposes of spending.

**By consensus, PATFOs mined in blocks prior to the activation of [Token-Aware Transaction Validation](#token-aware-transaction-validation) must remain unspendable after activation**. (Please note: the presence of PATFOs does not render a transaction invalid; until activation, valid blocks may contain PATFOs.)

After activation, any transaction creating an invalid token prefix<sup>2</sup> is itself invalid, and all transactions must pass [Token-Aware Transaction Validation](#token-aware-transaction-validation).

<details>

<summary>Notes</summary>

1. For pre-activation token-forgery outputs (PATFOs), this has been the case for even longer than `OP_RETURN` outputs: PATFOs have been provably unspendable since the Bitcoin Cash protocol's publication in 2008.
2. That is, any transaction output where locking bytecode index `0` is set to the `PREFIX_TOKEN` codepoint, but a valid token prefix cannot be parsed.

</details>

### Token-Aware Transaction Validation

For any transaction to be valid, the [**token validation algorithm**](#token-validation-algorithm) must succeed.

#### Token Validation Algorithm

Given the following **definitions**:

1. Reducing the set of UTXOs spent by the transaction:
   1. A key-value map of **`Available_Sums_By_Category`** (mapping category IDs to positive, 64-bit integers) is created by summing the input `amount` of each token category.
   2. A key-value map of **`Available_Mutable_Tokens_By_Category`** (mapping category IDs to positive integers) is created by summing the count of input mutable tokens for each category.
   3. A list of **`Genesis_Categories`** is created including the outpoint transaction hash of each input with an outpoint index of `0` (i.e. the spent UTXO was the 0th output in its transaction).
   4. A de-duplicated list of **`Input_Minting_Categories`** is created including the category ID of each input minting token.
   5. A list of **`Available_Minting_Categories`** is the combination of `Genesis_Categories` and `Input_Minting_Categories`.
   6. A list of all **`Available_Immutable_Tokens`** (including duplicates) is created including each NFT which **does not** have a `minting` or `mutable` capability.
2. Reducing the set of outputs created by the transaction:
   1. A key-value map of **`Output_Sums_By_Category`** (mapping category IDs to positive, 64-bit integers) is created by summing the output `amount` of each token category.
   2. A key-value map of **`Output_Mutable_Tokens_By_Category`** (mapping category IDs to positive integers) is created by summing the count of output mutable tokens for each category.
   3. A de-duplicated list of **`Output_Minting_Categories`** is created including the category ID of each output minting token.
   4. A list of all **`Output_Immutable_Tokens`** (including duplicates) is created including each NFT which **does not** have a `minting` or `mutable` capability.

Perform the following **validations**:

1. Each category in `Output_Minting_Categories` must exist in `Available_Minting_Categories`.
2. Each category in `Output_Sums_By_Category` must either:
   1. Have an equal or greater sum in `Available_Sums_By_Category`, or
   2. Exist in `Genesis_Categories` and have an output sum no greater than `9223372036854775807` (the maximum VM number).
3. For each category in `Output_Mutable_Tokens_By_Category`, if the token's category ID exists in `Available_Minting_Categories`, skip this (valid) category. Else:
   1. Deduct the sum in `Output_Mutable_Tokens_By_Category` from the sum available in `Available_Mutable_Tokens_By_Category`. If the value falls below `0`, **fail validation**.
4. For each token in `Output_Immutable_Tokens`, if the token's category ID exists in `Available_Minting_Categories`, skip this (valid) token. Else:
   1. If an equivalent token exists in `Available_Immutable_Tokens` (comparing both category ID and commitment), remove it and continue to the next token. Else:
      1. Deduct `1` from the sum available for the token's category in `Available_Mutable_Tokens_By_Category`. If no mutable tokens are available to downgrade, **fail validation**.

Note: because coinbase transactions have only one input with an outpoint index of `4294967295`, coinbase transactions can never include a token prefix in any output.

See [Implementations](#implementations) for examples of this algorithm in multiple programming languages.

### Token Inspection Operations

The following 6 operations pop the top item from the stack as an index (VM Number) and push a single result to the stack. If the consumed value is not a valid, minimally-encoded index for the operation, an error is produced.

| Name                       | Codepoint      | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| -------------------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OP_UTXOTOKENCATEGORY`     | `0xce` (`206`) | Pop the top item from the stack as an input index (VM Number). If the Unspent Transaction Output (UTXO) spent by that input includes no tokens, push a 0 (VM Number) to the stack. If the UTXO does not include a non-fungible token with a capability, push the UTXO's token category, otherwise, push the concatenation of the token category and capability, where the mutable capability is represented by 1 (VM Number) and the minting capability is represented by 2 (VM Number). |
| `OP_UTXOTOKENCOMMITMENT`   | `0xcf` (`207`) | Pop the top item from the stack as an input index (VM Number). Push the token commitment of the Unspent Transaction Output (UTXO) spent by that input to the stack. If the UTXO does not include a non-fungible token, or if it includes a non-fungible token with a zero-length commitment, push a 0 (VM Number).                                                                                                                                                                       |
| `OP_UTXOTOKENAMOUNT`       | `0xd0` (`208`) | Pop the top item from the stack as an input index (VM Number). Push the fungible token amount of the Unspent Transaction Output (UTXO) spent by that input to the stack as a VM Number. If the UTXO includes no fungible tokens, push a 0 (VM Number).                                                                                                                                                                                                                                   |
| `OP_OUTPUTTOKENCATEGORY`   | `0xd1` (`209`) | Pop the top item from the stack as an output index (VM Number). If the output at that index includes no tokens, push a 0 (VM Number) to the stack. If the output does not include a non-fungible token with a capability, push the output's token category, otherwise, push the concatenation of the token category and capability, where the mutable capability is represented by 1 (VM Number) and the minting capability is represented by 2 (VM Number).                             |
| `OP_OUTPUTTOKENCOMMITMENT` | `0xd2` (`210`) | Pop the top item from the stack as an output index (VM Number). Push the token commitment of the output at that index to the stack. If the output does not include a non-fungible token, or if it includes a non-fungible token with a zero-length commitment, push a 0 (VM Number).                                                                                                                                                                                                     |
| `OP_OUTPUTTOKENAMOUNT`     | `0xd3` (`211`) | Pop the top item from the stack as an output index (VM Number). Push the fungible token amount of the output at that index to the stack as a VM Number. If the output includes no fungible tokens, push a 0 (VM Number).                                                                                                                                                                                                                                                                 |

<details>

<summary><strong>Token Inspection Operation Test Vectors</strong></summary>

The following test vectors demonstrate the expected result of each token inspection operation for a particular output. The token category ID is `0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb` and commitments use repetitions of `0xcc`.

For the complete set of test vectors, see [Test Vectors](#test-vectors).

| Description                                        | `OP_UTXOTOKENCATEGORY`/`OP_OUTPUTTOKENCATEGORY` (Hex)                | `OP_UTXOTOKENCOMMITMENT`/`OP_OUTPUTTOKENCOMMITMENT` (Hex)                          | `OP_UTXOTOKENAMOUNT`/`OP_OUTPUTTOKENAMOUNT` (Hex) |
| -------------------------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------- |
| no NFT; 0 fungible                                 | (empty item)                                                         | (empty item)                                                                       | (empty item)                                      |
| no NFT; 1 fungible                                 | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | (empty item)                                                                       | `01`                                              |
| no NFT; 252 fungible                               | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | (empty item)                                                                       | `fc`                                              |
| no NFT; 253 fungible                               | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | (empty item)                                                                       | `fd00`                                            |
| no NFT; 9223372036854775807 fungible               | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | (empty item)                                                                       | `ffffffffffffff7f`                                |
| 0-byte immutable NFT; 0 fungible                   | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | (empty item)                                                                       | (empty item)                                      |
| 0-byte immutable NFT; 1 fungible                   | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | (empty item)                                                                       | `01`                                              |
| 0-byte immutable NFT; 253 fungible                 | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | (empty item)                                                                       | `fd00`                                            |
| 0-byte immutable NFT; 9223372036854775807 fungible | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | (empty item)                                                                       | `ffffffffffffff7f`                                |
| 1-byte immutable NFT; 252 fungible                 | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | `cc`                                                                               | `fc`                                              |
| 2-byte immutable NFT; 253 fungible                 | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | `cccc`                                                                             | `fd00`                                            |
| 10-byte immutable NFT; 65535 fungible              | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | `cccccccccccccccccccc`                                                             | `ffff`                                            |
| 40-byte immutable NFT; 65536 fungible              | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb`   | `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` | `00000100`                                        |
| 0-byte, mutable NFT; 0 fungible                    | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb01` | (empty item)                                                                       | (empty item)                                      |
| 0-byte, mutable NFT; 4294967295 fungible           | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb01` | (empty item)                                                                       | `ffffffff`                                        |
| 1-byte, mutable NFT; 4294967296 fungible           | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb01` | `cc`                                                                               | `0000000001000000`                                |
| 2-byte, mutable NFT; 9223372036854775807 fungible  | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb01` | `cccc`                                                                             | `ffffffffffffff7f`                                |
| 10-byte, mutable NFT; 1 fungible                   | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb01` | `cccccccccccccccccccc`                                                             | `01`                                              |
| 40-byte, mutable NFT; 252 fungible                 | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb01` | `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` | `fc`                                              |
| 0-byte, minting NFT; 0 fungible                    | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb02` | (empty item)                                                                       | (empty item)                                      |
| 0-byte, minting NFT; 253 fungible                  | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb02` | (empty item)                                                                       | `fd00`                                            |
| 1-byte, minting NFT; 65535 fungible                | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb02` | `cc`                                                                               | `ffff`                                            |
| 2-byte, minting NFT; 65536 fungible                | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb02` | `cccc`                                                                             | `00000100`                                        |
| 10-byte, minting NFT; 4294967297 fungible          | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb02` | `cccccccccccccccccccc`                                                             | `0100000001000000`                                |
| 40-byte, minting NFT; 9223372036854775807 fungible | `bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb02` | `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` | `ffffffffffffff7f`                                |

</details>

#### Existing Introspection Operations

Note, this specification has **no impact on the behavior of [`OP_UTXOBYTECODE`, `OP_ACTIVEBYTECODE`, or `OP_OUTPUTBYTECODE`](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Add-Native-Introspection-Opcodes.md)**. Each operation continues to return only the contents of the respective bytecode, excluding both the `CompactSize`-encoded "`token_prefix_and_locking_bytecode_length`" and the token prefix (if present).

#### Interpretation of Signature Preimage Inspection

It is possible to design contracts which inefficiently inspect the encoding of tokens using **signature preimage inspection** – inspecting the contents of a preimage for which a signature passes both `OP_CHECKSIG(VERIFY)` and `OP_CHECKDATASIG(VERIFY)`.

This specification interprets all signature preimage inspection of tokens as **intentional**: these constructions are designed to succeed or fail based on the encoding of the signature preimage, and they can be used (by design) to test for 1) the availability of some types of proposed-but-not-activated upgrades, and/or 2) a contracts' presence on a fork of Bitcoin Cash. This notice codifies a network policy: the possible existence of these contracts will not preclude future upgrades from adding additional output prefix or transaction formats. (The security of a contract is the responsibility of the entity locking funds in that contract; funds can always be locked in insecure contracts, e.g. `OP_DROP OP_1`.)

Contract authors are advised to use [Token Inspection Operations](#token-inspection-operations) for all constructions intended to inspect the actual properties of tokens within a transaction.

### Signing Serialization of Tokens

The [signing serialization algorithm](https://github.com/bitcoincashorg/bitcoincash.org/blob/3e2e6da8c38dab7ba12149d327bc4b259aaad684/spec/replay-protected-sighash.md) (A.K.A `SIGHASH` algorithm) is enhanced to support tokens: when evaluating a UTXO that includes tokens, the full, [encoded token prefix](#token-encoding) (including `PREFIX_TOKEN`) must be included immediately before the `coveredBytecode` (A.K.A. `scriptCode`). Note: this behavior applies for all signing serialization types in the evaluation; it does not require a signing serialization type/flag.

### `SIGHASH_UTXOS`

A new signing serialization type, `SIGHASH_UTXOS`, is defined at `0x20` (`32`/`0b100000`). When `SIGHASH_UTXOS` is enabled, `hashUtxos` is inserted in the [signing serialization algorithm](https://github.com/bitcoincashorg/bitcoincash.org/blob/3e2e6da8c38dab7ba12149d327bc4b259aaad684/spec/replay-protected-sighash.md) immediately following `hashPrevouts`. `hashUtxos` is a 32-byte double SHA256 of the serialization of all UTXOs spent by the transaction's inputs, concatenated in input order, excluding output count. (Note: this serialization is equivalent to the segment of a P2P transaction message beginning after `output count` and ending before `locktime` if the UTXOs were serialized in order as the transaction's outputs.)

The `SIGHASH_UTXOS` and `SIGHASH_ANYONECANPAY` types must not be used together; if a signature in which both flags are enabled is encountered during VM evaluation, an error is emitted (evaluation fails).

The `SIGHASH_UTXOS` type must be used with the `SIGHASH_FORKID` type; if a signature is encountered during VM evaluation with the `SIGHASH_UTXOS` flag and without the `SIGHASH_FORKID` flag, an error is emitted (evaluation fails).

**For security, wallets should enable `SIGHASH_UTXOS` when participating in multi-entity transactions**. This includes both 1) transactions where signatures are collected from multiple keys and assembled into a single transaction, and 2) transactions involving contracts that can be influenced by multiple entities (e.g. covenants). (See [Recommendation of `SIGHASH_UTXOS` for Multi-Entity Transactions](rationale.md#recommendation-of-sighashutxos-for-multi-entity-transactions).)

### Double Spend Proof Support

Transactions employing `SIGHASH_UTXOS` and transactions spending outputs containing tokens are not protected by either of the [beta specifications for Double Spend Proofs (DSP)](https://documentation.cash/protocol/network/messages/dsproof-beta). Support for these features are left to future proposals.

### CashAddress Token Support

Two new [`CashAddress` types](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/cashaddr.md#version-byte) are specified to indicate support for accepting tokens:

| Type Bits      | Meaning           |
| -------------- | ----------------- |
| `2` (`0b0010`) | Token-Aware P2PKH |
| `3` (`0b0011`) | Token-Aware P2SH  |

**Token-aware wallet software** – wallet software which supports management of tokens – may use these CashAddress version byte values to signal token support.

Token-aware wallet software **must refuse to send tokens to addresses without explicit token support** i.e. `P2PKH` CashAddresses (type bits: `0b0000`), `P2SH` CashAddresses (type bits: `0b0001`), and legacy Base58 addresses.

<details>

<summary><strong>Token-Aware CashAddress Test Vectors</strong></summary>

Test vectors for the CashAddress format have been [standardized and widely used since 2017](https://github.com/bitcoincashorg/bitcoincash.org/blob/3e2e6da8c38dab7ba12149d327bc4b259aaad684/spec/cashaddr.md), including test vectors for not-yet-defined `type` values.

While this specification simply uses the available `type` values, to assist implementers, the below test vectors have been added to the existing CashAddress test vectors in [`test-vectors/cashaddr.json`](./test-vectors/cashaddr.json). (For details, see [Test Vectors](#test-vectors).)

#### Token-Aware CashAddresses

| CashAddress                                                                 | Type Bits               | Size Bits      | Payload (Hex)                                                      |
| --------------------------------------------------------------------------- | ----------------------- | -------------- | ------------------------------------------------------------------ |
| `bitcoincash:qr7fzmep8g7h7ymfxy74lgc0v950j3r2959lhtxxsl`                    | `0` (P2PKH)             | `0` (20 bytes) | `fc916f213a3d7f1369313d5fa30f6168f9446a2d`                         |
| `bitcoincash:zr7fzmep8g7h7ymfxy74lgc0v950j3r295z4y4gq0v`                    | `2` (Token-Aware P2PKH) | `0` (20 bytes) | `fc916f213a3d7f1369313d5fa30f6168f9446a2d`                         |
| `bchtest:qr7fzmep8g7h7ymfxy74lgc0v950j3r295pdnvy3hr`                        | `0` (P2PKH)             | `0` (20 bytes) | `fc916f213a3d7f1369313d5fa30f6168f9446a2d`                         |
| `bchtest:zr7fzmep8g7h7ymfxy74lgc0v950j3r295x8qj2hgs`                        | `2` (Token-Aware P2PKH) | `0` (20 bytes) | `fc916f213a3d7f1369313d5fa30f6168f9446a2d`                         |
| `bchreg:qr7fzmep8g7h7ymfxy74lgc0v950j3r295m39d8z59`                         | `0` (P2PKH)             | `0` (20 bytes) | `fc916f213a3d7f1369313d5fa30f6168f9446a2d`                         |
| `bchreg:zr7fzmep8g7h7ymfxy74lgc0v950j3r295umknfytk`                         | `2` (Token-Aware P2PKH) | `0` (20 bytes) | `fc916f213a3d7f1369313d5fa30f6168f9446a2d`                         |
| `prefix:qr7fzmep8g7h7ymfxy74lgc0v950j3r295fu6e430r`                         | `0` (P2PKH)             | `0` (20 bytes) | `fc916f213a3d7f1369313d5fa30f6168f9446a2d`                         |
| `prefix:zr7fzmep8g7h7ymfxy74lgc0v950j3r295wkf8mhss`                         | `2` (Token-Aware P2PKH) | `0` (20 bytes) | `fc916f213a3d7f1369313d5fa30f6168f9446a2d`                         |
| `bitcoincash:qpagr634w55t4wp56ftxx53xukhqgl24yse53qxdge`                    | `0` (P2PKH)             | `0` (20 bytes) | `7a81ea357528bab834d256635226e5ae047d5524`                         |
| `bitcoincash:zpagr634w55t4wp56ftxx53xukhqgl24ys77z7gth2`                    | `2` (Token-Aware P2PKH) | `0` (20 bytes) | `7a81ea357528bab834d256635226e5ae047d5524`                         |
| `bitcoincash:qq9l9e2dgkx0hp43qm3c3h252e9euugrfc6vlt3r9e`                    | `0` (P2PKH)             | `0` (20 bytes) | `0bf2e54d458cfb86b106e388dd54564b9e71034e`                         |
| `bitcoincash:zq9l9e2dgkx0hp43qm3c3h252e9euugrfcaxv4l962`                    | `2` (Token-Aware P2PKH) | `0` (20 bytes) | `0bf2e54d458cfb86b106e388dd54564b9e71034e`                         |
| `bitcoincash:qre24q38ghy6k3pegpyvtxahu8q8hqmxmqqn28z85p`                    | `0` (P2PKH)             | `0` (20 bytes) | `f2aa822745c9ab44394048c59bb7e1c07b8366d8`                         |
| `bitcoincash:zre24q38ghy6k3pegpyvtxahu8q8hqmxmq8eeevptj`                    | `2` (Token-Aware P2PKH) | `0` (20 bytes) | `f2aa822745c9ab44394048c59bb7e1c07b8366d8`                         |
| `bitcoincash:qz7xc0vl85nck65ffrsx5wvewjznp9lflgktxc5878`                    | `0` (P2PKH)             | `0` (20 bytes) | `bc6c3d9f3d278b6a8948e06a399974853097e9fa`                         |
| `bitcoincash:zz7xc0vl85nck65ffrsx5wvewjznp9lflg3p4x6pp5`                    | `2` (Token-Aware P2PKH) | `0` (20 bytes) | `bc6c3d9f3d278b6a8948e06a399974853097e9fa`                         |
| `bitcoincash:ppawqn2h74a4t50phuza84kdp3794pq3ccvm92p8sh`                    | `1` (P2SH)              | `0` (20 bytes) | `7ae04d57f57b55d1e1bf05d3d6cd0c7c5a8411c6`                         |
| `bitcoincash:rpawqn2h74a4t50phuza84kdp3794pq3cct3k50p0y`                    | `3` (Token-Aware P2SH)  | `0` (20 bytes) | `7ae04d57f57b55d1e1bf05d3d6cd0c7c5a8411c6`                         |
| `bitcoincash:pqv53dwyatxse2xh7nnlqhyr6ryjgfdtagkd4vc388`                    | `1` (P2SH)              | `0` (20 bytes) | `1948b5c4eacd0ca8d7f4e7f05c83d0c92425abea`                         |
| `bitcoincash:rqv53dwyatxse2xh7nnlqhyr6ryjgfdtag38xjkhc5`                    | `3` (Token-Aware P2SH)  | `0` (20 bytes) | `1948b5c4eacd0ca8d7f4e7f05c83d0c92425abea`                         |
| `bitcoincash:prseh0a4aejjcewhc665wjqhppgwrz2lw5txgn666a`                    | `1` (P2SH)              | `0` (20 bytes) | `e19bbfb5ee652c65d7c6b54748170850e1895f75`                         |
| `bitcoincash:rrseh0a4aejjcewhc665wjqhppgwrz2lw5vvmd5u9w`                    | `3` (Token-Aware P2SH)  | `0` (20 bytes) | `e19bbfb5ee652c65d7c6b54748170850e1895f75`                         |
| `bitcoincash:pzltaslh7xnrsxeqm7qtvh0v53n3gfk0v5wwf6d7j4`                    | `1` (P2SH)              | `0` (20 bytes) | `bebec3f7f1a6381b20df80b65deca4671426cf65`                         |
| `bitcoincash:rzltaslh7xnrsxeqm7qtvh0v53n3gfk0v5fy6yrcdx`                    | `3` (Token-Aware P2SH)  | `0` (20 bytes) | `bebec3f7f1a6381b20df80b65deca4671426cf65`                         |
| `bitcoincash:pvqqqqqqqqqqqqqqqqqqqqqqzg69v7ysqqqqqqqqqqqqqqqqqqqqqpkp7fqn0` | `1` (P2SH)              | `3` (32 bytes) | `0000000000000000000000000000123456789000000000000000000000000000` |
| `bitcoincash:rvqqqqqqqqqqqqqqqqqqqqqqzg69v7ysqqqqqqqqqqqqqqqqqqqqqn9alsp2y` | `3` (Token-Aware P2SH)  | `3` (32 bytes) | `0000000000000000000000000000123456789000000000000000000000000000` |
| `bitcoincash:pdzyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3jh2p5nn` | `1` (P2SH)              | `3` (32 bytes) | `4444444444444444444444444444444444444444444444444444444444444444` |
| `bitcoincash:rdzyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygrpttc42c` | `3` (Token-Aware P2SH)  | `3` (32 bytes) | `4444444444444444444444444444444444444444444444444444444444444444` |
| `bitcoincash:pwyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygsh3sujgcr` | `1` (P2SH)              | `3` (32 bytes) | `8888888888888888888888888888888888888888888888888888888888888888` |
| `bitcoincash:rwyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs9zvatfpg` | `3` (Token-Aware P2SH)  | `3` (32 bytes) | `8888888888888888888888888888888888888888888888888888888888888888` |
| `bitcoincash:p0xvenxvenxvenxvenxvenxvenxvenxvenxvenxvenxvenxvenxvcm6gz4t77` | `1` (P2SH)              | `3` (32 bytes) | `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` |
| `bitcoincash:r0xvenxvenxvenxvenxvenxvenxvenxvenxvenxvenxvenxvenxvcff5rv284` | `3` (Token-Aware P2SH)  | `3` (32 bytes) | `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` |
| `bitcoincash:p0llllllllllllllllllllllllllllllllllllllllllllllllll7x3vthu35` | `1` (P2SH)              | `3` (32 bytes) | `ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff` |
| `bitcoincash:r0llllllllllllllllllllllllllllllllllllllllllllllllll75zs2wagl` | `3` (Token-Aware P2SH)  | `3` (32 bytes) | `ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff` |

</details>

### Token-Aware BIP69 Sorting Algorithm

The [BIP69 transaction output sorting algorithm](https://github.com/bitcoin/bips/blob/master/bip-0069.mediawiki#transaction-outputs) is extended to support the sorting of outputs by token information.

As with the existing algorithm, the additional fields are ordered for sorting efficiency:

1. Output value – ascending
2. locking bytecode – sorted lexicographically<sup>1</sup>, ascending (e.g. `0x51` < `0x5161`)
3. token information: (no tokens < has tokens)
   1. amount – ascending (e.g. `0` < `1`)
   2. `HAS_NFT` – `false` < `true`
      1. capability – ascending; `none` < `mutable` < `minting`
      2. commitment – sorted lexicographically, ascending, short-to-long; e.g. zero-length < `0x00` < `0x01` < `0x0100`)
   3. category – sorted lexicographically, ascending, where bytes are in little-endian order (matching the order used in encoded transactions)

**Notes**

1. For clarity, lexicographically-sorted fields must be compared byte-by-byte from start to end. Where one item is a prefix of another item, the shorter item is considered the "lower" value. E.g. the following values are sorted lexicographically in ascending order: `0x00`, `0x0011`, `0x11`, `0x1100`.

### Fungible Token Supply Definitions

Several measurements of fungible token supply are standardized for wider ecosystem compatibility. (See [Specification of Token Supply Definitions](rationale.md#specification-of-token-supply-definitions).)

By design, [Genesis Supply](#genesis-supply), [Reserved Supply](#reserved-supply), [Circulating Supply](#circulating-supply), and [Total Supply](#total-supply) of any fungible token category will not exceed `9223372036854775807` (the maximum VM number).

#### Genesis Supply

A token category's **genesis supply** is an **immutable, easily-computed, maximum possible supply**, known since the token category's genesis transaction. It overestimates total supply if any amount of tokens have been destroyed since the genesis transaction.

The genesis supply of a fungible token category can be computed by parsing the outputs of the category's genesis transaction and summing the `amount` of fungible tokens matching the category's ID.

#### Total Supply

A token category's **total supply** is the sum – at a particular moment in time – of **tokens which are either in circulation or may enter circulation in the future**. A token category's total supply is always less than or equal to its genesis supply.

The total supply of a fungible token category can be computed by retrieving all UTXOs which contain token prefixes matching the category ID, removing provably-destroyed outputs (spent to `OP_RETURN` outputs), and summing the remaining `amount`s.

Software implementations should emphasize total supply in user interfaces for token categories which do not meet the requirements for emphasizing [circulating supply](#circulating-supply).

#### Reserved Supply

A token category's **reserved supply** or **unissued supply** is the sum – at a particular moment in time – of tokens held in reserve by the issuing entity. **This is the portion of the supply which the issuer represents as "not in circulation".**

The reserved supply of a fungible token category can be computed by retrieving all UTXOs which contain token prefixes matching the category ID, removing provably-destroyed outputs (spent to `OP_RETURN` outputs), and summing the `amount`s held in prefixes which have either the `minting` or `mutable` capability.

#### Circulating Supply

A token category's **circulating supply** is the sum – at a particular moment in time – of tokens not held in reserve by the issuing entity. **This is the portion of the supply which the issuer represents as "in circulation".**

The **circulating supply** of a fungible token category can be computed by subtracting the [reserved supply](#reserved-supply) from the [total supply](#total-supply).

Software implementations might choose to emphasize circulating supply (rather than total supply) in user interfaces for token categories which:

- are issued by an entity trusted by the user, or
- are issued by a covenant (of a construction known to the verifier) for which token issuance is limited (via a strategy trusted by the user).

## Usage Examples

- [Appendix: Usage Examples &rarr;](examples.md#usage-examples)
  - [Identity Tokens](examples.md#identity-tokens)
  - [Covenant-Tracking Identity Tokens](examples.md#covenant-tracking-identity-tokens)
  - [Depository Child Covenants](examples.md#depository-child-covenants)
  - [Voting with Fungible Tokens](examples.md#voting-with-fungible-tokens)
    - [Sealed Voting](examples.md#sealed-voting)
  - [Multithreaded Covenants](examples.md#multithreaded-covenants)
  - [Multi-Covenant, Decentralized Applications](examples.md#multi-covenant-decentralized-applications)

## Rationale

- [Appendix: Rationale &rarr;](rationale.md#rationale)
  - [Incompatibility of Token Fungibility and Token Commitments](rationale.md#incompatibility-of-token-fungibility-and-token-commitments)
  - [Selection of Commitment Types](rationale.md#selection-of-commitment-types)
    - [Boolean Commitment Type](rationale.md#boolean-commitment-type)
    - [Bitfield Commitment Type](rationale.md#bitfield-commitment-type)
    - [Hash Commitment Types](rationale.md#hash-commitment-types)
      - [Secret Hash Commitment Type](rationale.md#secret-hash-commitment-type)
      - [Nonsecret Hash Commitment Type](rationale.md#nonsecret-hash-commitment-type)
    - [Public Key Commitment Type](rationale.md#public-key-commitment-type)
    - [Signature Commitment Type](rationale.md#signature-commitment-type)
  - [Shared Codepoint for All Tokens](rationale.md#shared-codepoint-for-all-tokens)
  - [Behavior of Minting and Mutable Tokens](rationale.md#behavior-of-minting-and-mutable-tokens)
  - [Exclusion of Cloneable Capability](rationale.md#exclusion-of-cloneable-capability)
  - [Avoiding Proof-of-Work for Token Data Compression](rationale.md#avoiding-proof-of-work-for-token-data-compression)
  - [Use of Transaction IDs as Token Category IDs](rationale.md#use-of-transaction-ids-as-token-category-ids)
  - [One Token Prefix Per Output](rationale.md#one-token-prefix-per-output)
  - [Including Capabilities in Token Category Inspection Operations](rationale.md#including-capabilities-in-token-category-inspection-operations)
  - [Support for Zero-Length Commitments](rationale.md#support-for-zero-length-commitments)
  - [Limitation of Non-Fungible Token Commitment Length](rationale.md#limitation-of-non-fungible-token-commitment-length)
  - [Inclusion of Token-Aware CashAddresses](rationale.md#inclusion-of-token-aware-cashaddresses)
  - [Recommendation of `SIGHASH_UTXOS` for Multi-Entity Transactions](rationale.md#recommendation-of-sighash_utxos-for-multi-entity-transactions)
  - [Limitation of Fungible Token Supply](rationale.md#limitation-of-fungible-token-supply)
  - [Specification of Token Supply Definitions](rationale.md#specification-of-token-supply-definitions)

## Prior Art & Alternatives

- [Appendix: Prior Art & Alternatives &rarr;](alternatives.md#prior-art--alternatives)
  - [PMv3](alternatives.md#pmv3)
  - [Bitauth](alternatives.md#bitauth)
  - [Colored Coins](alternatives.md#colored-coins)
    - [OP_CHECKCOLORVERIFY](alternatives.md#op_checkcolorverify)
    - [Freimarkets](alternatives.md#freimarkets)
    - [Confidential Assets](alternatives.md#confidential-assets)
    - [OP_GROUP](alternatives.md#op_group)
    - [Group Tokenization](alternatives.md#group-tokenization)
    - [Simple Ledger Protocol (v1)](alternatives.md#simple-ledger-protocol-v1)
    - [Unforgeable Groups](alternatives.md#unforgeable-groups)

## Test Vectors

Sets of cross-implementation test vectors are provided in the [`test-vectors`](./test-vectors/) directory. Each set is described below.

### Token-Aware CashAddress Test Vectors

[`cashaddr.json`](./test-vectors/cashaddr.json) includes an updated set of test vectors including tests for [token-aware CashAddresses](#cashaddress-token-support).

Test vectors for the CashAddress format have been [standardized and widely used since 2017](https://github.com/bitcoincashorg/bitcoincash.org/blob/3e2e6da8c38dab7ba12149d327bc4b259aaad684/spec/cashaddr.md), including test vectors for not-yet-defined `type` values.

While this specification simply uses the available `type` values, to assist implementers, several additional test vectors have been added to the existing CashAddress test vectors in [`test-vectors/cashaddr.json`](./test-vectors/cashaddr.json).

### Token Encoding Test Vectors

A complete set of test vectors that validate token encoding can be found in [`test-vectors/token-prefix-valid.json`](./test-vectors/token-prefix-valid.json) and [`test-vectors/token-prefix-invalid.json`](./test-vectors/token-prefix-invalid.json), respectively.

### Transaction Validation Test Vectors

The [`test-vectors/vmb_tests`](./test-vectors/vmb_tests/) directory contains sets of transaction test vectors that validate all technical elements of this proposal.

To maximize portability between implementations, these test vectors use Libauth's [full-transaction testing strategy](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/vmb-tests/readme.md). Test vectors are sorted into files based on their expected behavior:

- **Pre-activation test vectors** – these vectors test transaction validation prior to the activation of this proposal.
  - [`bch_vmb_tests_before_chip_cashtokens_invalid.json`](./test-vectors/vmb_tests/bch_vmb_tests_before_chip_cashtokens_invalid.json) - test vectors that must fail validation in both nonstandard and standard mode (see [Standard Vs. Non-Standard VMs](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/vmb-tests/readme.md#standard-vs-non-standard-vms)). To assist implementers, a companion [`reasons` file](./test-vectors/vmb_tests/bch_vmb_tests_before_chip_cashtokens_invalid_reasons.json) describes the reason each test vector is expected to fail.
  - [`bch_vmb_tests_before_chip_cashtokens_nonstandard.json`](./test-vectors/vmb_tests/bch_vmb_tests_before_chip_cashtokens_nonstandard.json) - test vectors that must fail validation in standard mode but pass validation in nonstandard mode. A companion [`reasons` file](./test-vectors/vmb_tests/bch_vmb_tests_before_chip_cashtokens_nonstandard_reasons.json) describes the reason each test vector is expected to fail in standard mode.
  - [`bch_vmb_tests_before_chip_cashtokens_standard.json`](./test-vectors/vmb_tests/bch_vmb_tests_before_chip_cashtokens_standard.json) - test vectors that must pass validation in both standard and nonstandard mode.
- **Post-activation test vectors** – these vectors test transaction validation as it must behave after activation of this proposal.
  - [`bch_vmb_tests_chip_cashtokens_invalid.json`](./test-vectors/vmb_tests/bch_vmb_tests_chip_cashtokens_invalid.json) - test vectors that must fail validation in both nonstandard and standard mode. To assist implementers, a companion [`reasons` file](./test-vectors/vmb_tests/bch_vmb_tests_chip_cashtokens_invalid_reasons.json) describes the reason each test vector is expected to fail.
  - [`bch_vmb_tests_chip_cashtokens_nonstandard.json`](./test-vectors/vmb_tests/bch_vmb_tests_chip_cashtokens_nonstandard.json) - test vectors that must fail validation in standard mode but pass validation in nonstandard mode. A companion [`reasons` file](./test-vectors/vmb_tests/bch_vmb_tests_chip_cashtokens_nonstandard_reasons.json) describes the reason each test vector is expected to fail in standard mode.
  - [`bch_vmb_tests_chip_cashtokens_standard.json`](./test-vectors/vmb_tests/bch_vmb_tests_chip_cashtokens_standard.json) - test vectors that must pass validation in both standard and nonstandard mode.

Each test vector is an array including:

1. A short, unique identifier for the test (based on the hash of the test contents)
2. A string describing the purpose/behavior of the test
3. The unlocking script under test (disassembled, i.e. human-readable)
4. The locking script under test (disassembled)
5. The full, encoded test transaction
6. An encoded list of unspent transaction outputs (UTXOs) with which to verify the test transaction (ordered to match the input order of the test transaction)

**Only array items 5 and 6 are strictly necessary**; items 1 through 4 are purely informational, and may be useful in debugging and cross-implementation communication.

To use these test vectors, implementations should decode the transaction under test (5) and its UTXOs (6), then validate the transaction using all of the implementation's transaction validation infrastructure initialized in the expected standard/nonstandard mode(s). See [Implementations](#implementations) for examples.

## Implementations

Please see the following implementations for additional examples and test vectors:

- C++
  - [Bitcoin Cash Node (BCHN)](https://bitcoincashnode.org/) – A professional, miner-friendly node that solves practical problems for Bitcoin Cash.
    - CashTokens support: [Merge Request !1580](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1580)
    - Token-aware CashAddresses: [Merge Request !1596](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1596)
    - This CHIP also includes CashToken integration test vectors for other proposals:
      - P2SH32: [Merge Request !1556](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1556)
      - 65-byte TXs: [Merge Request !1598](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1598)
- JavaScript/TypeScript
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash.
    - [Pull Request #98](https://github.com/bitauth/libauth/pull/98)
      - [Token encoding](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/message/transaction-encoding.ts#L395-L435)
      - [Token decoding](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/message/transaction-encoding.ts#L209-L324)
      - [Token-aware validation](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/vm/instruction-sets/bch/2023/bch-2023-tokens.ts#L163-L297)
      - [Token inspection operations](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/vm/instruction-sets/bch/2023/bch-2023-tokens.ts#L355-L389)
      - [Token-aware signing serialization](https://github.com/bitauth/libauth/blob/9dfa6cc0b8710dedfe007b47bd018f5a47079df5/src/lib/vm/instruction-sets/common/signing-serialization.ts#L456-L474)
      - [Token-aware CashAddresses](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/address/cash-address.ts#L61-L82)
      - [Verifying vmb_tests](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/vmb-tests/bch-vmb-tests.spec.ts#L122-L152)

## Stakeholder Responses & Statements

[Stakeholder Responses & Statements &rarr;](stakeholders.md)

## Feedback & Reviews

- [CashTokens CHIP Issues](https://github.com/bitjson/cashtokens/issues)
- [`CHIP 2022-02 CashTokens` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2022-02-cashtokens-token-primitives-for-bitcoin-cash/725)

## Acknowledgements

Thank you to the following contributors for reviewing and contributing improvements to this proposal, providing feedback, and promoting consensus among stakeholders:
[Calin Culianu](https://github.com/cculianu), [bitcoincashautist](https://github.com/A60AB5450353F40E), [imaginary_username](https://gitlab.com/im_uname), [Joshua Green](https://github.com/joshmg), [Andrew Groot](https://github.com/thesquaregroot), [Tom Zander](https://github.com/zander), [Andrew #128](https://gitlab.com/andrew-128), [Mathieu Geukens](https://github.com/mr-zwets), [Richard Brady](https://github.com/rnbrady), [Marty Alcala](https://github.com/msalcala11), [John Nieri](https://gitlab.com/emergent-reasons), [Jonathan Silverblood](https://gitlab.com/monsterbitar), [Benjamin Scherrey](https://github.com/scherrey), [Rosco Kalis](https://github.com/rkalis), [Deyan Dimitrov](https://github.com/dikel), [Jonas Lundqvist](https://github.com/jonas-lundqvist), [Joemar Taganna](https://github.com/joemarct), [Rucknium](https://github.com/Rucknium).

## Changelog

This section summarizes the evolution of this document.

- **v2.2.2 – 2022-5-20**
  - Mention unissued supply as an alternative term for reserved supply
  - Mark CHIP as final
- **v2.2.1 – 2022-11-15** ([`7552da2d`](https://github.com/bitjson/cashtokens/blob/7552da2dfad217aa5f4130f52d7d6cbbfeef7a23/readme.md))
  - Remove confusing recommendation about token-aware CashAddress usage ([#82](https://github.com/bitjson/cashtokens/issues/82))
  - Extract [`examples.md`](./examples.md), [`rationale.md`](./rationale.md), and [`alternatives.md`](./alternatives.md) for approachability
  - Add [`stakeholders.md`](./stakeholders.md) to collect final approvals
  - Expand test vectors ([#90](https://github.com/bitjson/cashtokens/pull/90))
- **v2.2.0 – 2022-9-30** ([`e02012a2`](https://github.com/bitjson/cashtokens/blob/e02012a219a0fb2abef02aa3e08ad326774bd3f3/readme.md))
  - Compress token encoding using bitfield ([#33](https://github.com/bitjson/cashtokens/pull/33))
  - Encode mutable capability as `0x01` and minting capability as `0x02`
  - Revert to limiting `commitment_length` to `40` bytes by consensus ([#23](https://github.com/bitjson/cashtokens/issues/23))
  - Revert `PREFIX_TOKEN` to a unique codepoint (`0xef`) ([#41](https://github.com/bitjson/cashtokens/issues/41))
  - Modify `OP_*TOKENCOMMITMENT` to push `0` for zero-length commitments ([#25](https://github.com/bitjson/cashtokens/issues/25))
  - Extend BIP69 sorting algorithm to support tokens ([#60](https://github.com/bitjson/cashtokens/pull/60))
  - Specify activation times
  - Expand test vectors
  - Note non-support of beta specs for double spend proofs
  - Improve rationale
- **v2.1.0 – 2022-6-30** ([`f8b500a0`](https://github.com/bitjson/cashtokens/blob/f8b500a051f82d42dbf9e9e890bc6cdc14592307/readme.md))
  - Expand motivation, benefits, rationale, prior art & alternatives
  - Simplify token encoding, update test vectors
  - Set `PREFIX_TOKEN` to `0xd0` and limit `commitment_length` using standardness
  - Specify handling of pre-activation token-forgery outputs
  - Specify token-aware signing serialization algorithm and `SIGHASH_UTXOS` ([#22](https://github.com/bitjson/cashtokens/issues/22))
- **v2.0.1 – 2022-2-25** ([`fcb110c3`](https://github.com/bitjson/cashtokens/blob/fcb110c3309901886b2c7d3417568d8b13fb01b5/readme.md))
  - Expand rationale
  - Note impossibility of valid token outputs in coinbase transactions
- **v2.0.0 – 2022-2-22** ([`879c55ed`](https://github.com/bitjson/cashtokens/blob/879c55edd7e9cd6a2c2d50990d89e5cc7cb07394/readme.md))
  - Initial publication (versioning begins at v2 to differentiate from [CashTokens v1](https://blog.bitjson.com/cashtokens-contract-validated-tokens-for-bitcoin-cash/))

## Copyright

This document is placed in the public domain.
