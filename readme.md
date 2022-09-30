# CHIP-2022-02-CashTokens: Token Primitives for Bitcoin Cash

        Title: Token Primitives for Bitcoin Cash
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2022-02-22
        Latest Revision Date: 2022-9-30
        Version: 2.2.0

## Summary

This proposal enables two new primitives on Bitcoin Cash: **fungible tokens** and **non-fungible tokens**.

### Terms

A **token** is an asset – distinct from the Bitcoin Cash currency – that can be created and transferred on the Bitcoin Cash network.

**Non-Fungible tokens (NFTs)** are a token type in which individual units cannot be merged or divided – each NFT contains a **commitment**, a short byte string attested to by the issuer of the NFT.

**Fungible tokens** are a token type in which individual units are undifferentiated – groups of fungible tokens can be freely divided and merged without tracking the identity of individual tokens (much like the Bitcoin Cash currency).

## Deployment

Deployment of this specification is proposed for the May 2023 upgrade.

- Activation is proposed for `1668513600` [MTP](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki), (`2022-11-15T12:00:00.000Z`) on `testnet4`, splitting to create `chipnet`.
- Activation is proposed for `1684152000` [MTP](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki), (`2023-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, and `testnet4`.

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

While many use cases for numeric commitments can be emulated with only byte-string commitments, a numeric primitive enables many contracts to reduce or offload state management altogether (e.g. [shareholder voting](#voting-with-fungible-tokens)), simplifying contract audits and reducing transaction sizes.

In this proposal, numeric commitments are called **fungible tokens**.

## Benefits

By enabling token primitives on Bitcoin Cash, this proposal offers several benefits.

### Cross-Contract Interfaces

Using **non-fungible tokens** (NFTs), contracts can **create messages that can be read by other contracts**. These messages are impersonation-proof: other contracts can safely read and act on the commitment, certain that it was produced by the claimed contract.

With contract interoperability, behavior can be broken into clusters of smaller, coordinating contracts, **reducing transaction sizes**. This interoperability further enables [covenants](#usage-examples) to communicate over **public interfaces**, allowing diverse ecosystems of compatible covenants to work together, even when developed and deployed separately.

Critically, this cross-contract interaction can be achieved within the "stateless" transaction model employed by Bitcoin Cash, rather than coordinating via shared global state. This allows Bitcoin Cash to support comparable contract functionality while retaining its [>1000x efficiency advantage](https://blog.bitjson.com/pmv3-build-decentralized-applications-on-bitcoin-cash/#stateless-network-stateful-covenants) in transaction and block validation.

### Decentralized Applications

Beyond enabling covenants to interoperate with other covenants, these token primitives allow for byte-efficient representations of complex internal state – supporting [advanced, decentralized applications](https://github.com/bitjson/jedex) on Bitcoin Cash.

**Non-fungible tokens** are critical for coordinating activity trustlessly between multiple covenants, enabling [covenant-tracking tokens](#covenant-tracking-identity-tokens), [depository child covenants](#depository-child-covenants), [multithreaded covenants](#multithreaded-covenants), and other constructions in which a particular covenant instance must be authenticated.

**Fungible tokens** are valuable for covenants to represent on-chain assets – e.g. voting shares, utility tokens, collateralized loans, prediction market options, etc. – and implement [complex coordination tasks](#voting-with-fungible-tokens) – e.g. liquidity-pooling, auctions, voting, sidechain withdrawals, spin-offs, mergers, and more.

### Universal Token Primitives

By exposing basic, consensus-validated token primitives, this proposal supports the development of higher-level, interoperable token standards (e.g. [SLP](https://slp.dev/)). Token primitives can be held by any contract, wallets can easily verify the authenticity of a token or group of tokens, and tokens cannot be inadvertently destroyed by wallet software that does not support tokens.

## Technical Summary

1. A token **category** can include both non-fungible and fungible tokens, and every category is represented by a 32-byte category identifier – the transaction ID of the outpoint spent to create the category.
   1. All fungible tokens for a category must be created when the token category is created, ensuring the total supply within a category remains below the maximum VM number.
   2. Non-fungible tokens may be created either at category creation or in later transactions that spend tokens with `minting` or `mutable` capabilities for that category.
2. Transaction outputs are extended to support four new `token` fields; every output can include one **non-fungible token** and any amount of **fungible tokens** from a single token category.
3. [Token inspection opcodes](#token-inspection-operations) allow contracts to operate on tokens, enabling [cross-contract interfaces and decentralized applications](#usage-examples).

### Transaction Output Data Model

This proposal extends the data model of transaction outputs to add four new `token` fields: token `category`, non-fungible token `capability`, non-fungible token `commitment`, and fungible token `amount`.

| Existing Fields  | Description                                                                                                                                                    |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Value            | The value of the output in satoshis, the smallest unit of bitcoin cash. (A.K.A. `vout`)                                                                        |
| Locking Bytecode | The VM bytecode used to encumber this transaction output. (A.K.A. `scriptPubKey`)                                                                              |
| **Token Fields** | (New, optional fields added by this proposal:)                                                                                                                 |
| Category         | The 32-byte ID of the token category to which the token(s) in this output belong. This field is omitted if no tokens are present.                              |
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

Token primitives are defined, token encoding and activation are specified, and six new token inspection opcodes are introduced. Transaction validation and transaction signing serialization is modified to support tokens, `SIGHASH_UTXOS` is specified, BIP69 sorting is extended to support tokens, and CashAddress `types` with token support are defined.

### Token Categories

Every token belongs to a **token category** specified via an immutable, 32-byte **Token Category ID** assigned in the category's **genesis transaction** – the transaction in which the token category is initially created.

Every token category ID is a transaction ID: the ID must be selected from the inputs of its genesis transaction, and only **token genesis inputs** – inputs which spend output `0` of their parent transaction – are eligible (i.e. outpoint transaction hashes of inputs with an outpoint index of `0`). As such, implementations can locate the genesis transaction of any category by identifying the transaction that spent the `0`th output of the transaction referenced by the category ID. (See [Use of Transaction IDs as Token Category IDs](#use-of-transaction-ids-as-token-category-ids).)

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

1. By consensus, `commitment_length` is limited to `40` (`0x28`), but future upgrades may increase this limit. Implementers are advised to ensure that values between `253` (`0xfdfd00`) and `65535` (`0xfdffff`) can be parsed. (See [Non-Fungible Token Commitment Length](#non-fungible-token-commitment-length).)
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

PATFOs are provably unspendable<sup>1</sup>; all software implementing this specification should immediately mark PATFOs as unspendable and/or remove them from their local view of the UTXO set (e.g. on startup and upon receipt).

**By consensus, PATFOs mined in blocks prior to the activation of [Token-Aware Transaction Validation](#token-aware-transaction-validation) must remain unspendable after activation**. (Please note: the presence of PATFOs does not render a transaction invalid; until activation, valid blocks may contain PATFOs.)

After activation, any transaction creating an invalid token prefix<sup>2</sup> is itself invalid, and all transactions must pass [Token-Aware Transaction Validation](#token-aware-transaction-validation).

<details>

<summary>Notes</summary>

1. For pre-activation token-forgery outputs (PATFOs), this has been the case for even longer than `OP_RETURN` outputs: PATFOs have been provably unspendable since the Bitcoin Cash protocol's publication in 2008. As such, they can be safely pruned without activation and at no risk of network consensus failures.
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
| `OP_OUTPUTTOKENCATEGORY`   | `0xd1` (`209`) | Pop the top item from the stack as an output index (VM Number). If the output spent by that input includes no tokens, push a 0 (VM Number) to the stack. If the output does not include a non-fungible token with a capability, push the output's token category, otherwise, push the concatenation of the token category and capability, where the mutable capability is represented by 1 (VM Number) and the minting capability is represented by 2 (VM Number).                       |
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

### Existing Introspection Operations

Note, this specification has **no impact on the behavior of [`OP_UTXOBYTECODE`, `OP_ACTIVEBYTECODE`, or `OP_OUTPUTBYTECODE`](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Add-Native-Introspection-Opcodes.md)**. Each operation continues to return only the contents of the respective bytecode, excluding both the `CompactSize`-encoded "`token_prefix_and_locking_bytecode_length`" and the token prefix (if present).

### Interpretation of Signature Preimage Inspection

It is possible to design contracts which inefficiently inspect the encoding of tokens using **signature preimage inspection** – inspecting the contents of a preimage for which a signature passes both `OP_CHECKSIG(VERIFY)` and `OP_CHECKDATASIG(VERIFY)`.

This specification interprets all signature preimage inspection of tokens as **intentional**: these constructions are designed to succeed or fail based on the encoding of the signature preimage, and they can be used (by design) to test for 1) the availability of some types of proposed-but-not-activated upgrades, and/or 2) a contracts' presence on a fork of Bitcoin Cash. This notice codifies a network policy: the possible existence of these contracts will not preclude future upgrades from adding additional output prefix or transaction formats. (The security of a contract is the responsibility of the entity locking funds in that contract; funds can always be locked in insecure contracts, e.g. `OP_DROP OP_1`.)

Contract authors are advised to use [Token Inspection Operations](#token-inspection-operations) for all constructions intended to inspect the actual properties of tokens within a transaction.

### Signing Serialization of Tokens

The [signing serialization algorithm](https://github.com/bitcoincashorg/bitcoincash.org/blob/3e2e6da8c38dab7ba12149d327bc4b259aaad684/spec/replay-protected-sighash.md) (A.K.A `SIGHASH` algorithm) is enhanced to support tokens: when evaluating a UTXO that includes tokens, the full, [encoded token prefix](#token-encoding) (including `PREFIX_TOKEN`) must be included immediately before the `coveredBytecode` (A.K.A. `scriptCode`). Note: this behavior applies for all signing serialization types in the evaluation; it does not require a signing serialization type/flag.

### `SIGHASH_UTXOS`

A new signing serialization type, `SIGHASH_UTXOS`, is defined at `0x20` (`32`/`0b100000`). When `SIGHASH_UTXOS` is enabled, `hashUtxos` is inserted in the [signing serialization algorithm](https://github.com/bitcoincashorg/bitcoincash.org/blob/3e2e6da8c38dab7ba12149d327bc4b259aaad684/spec/replay-protected-sighash.md) immediately following `hashPrevouts`. `hashUtxos` is a 32-byte double SHA256 of the serialization of all UTXOs spent by the transaction's inputs, concatenated in input order, excluding output count. (Note: this serialization is equivalent to the segment of a P2P transaction message beginning after `output count` and ending before `locktime` if the UTXOs were serialized in order as the transaction's outputs.)

The `SIGHASH_UTXOS` and `SIGHASH_ANYONECANPAY` types must not be used together; if a signature in which both flags are enabled is encountered during VM evaluation, an error is emitted (evaluation fails).

The `SIGHASH_UTXOS` type must be used with the `SIGHASH_FORKID` type; if a signature is encountered during VM evaluation with the `SIGHASH_UTXOS` flag and without the `SIGHASH_FORKID` flag, an error is emitted (evaluation fails).

**For security, wallets should enable `SIGHASH_UTXOS` when participating in multi-entity transactions**. This includes both 1) transactions where signatures are collected from multiple keys and assembled into a single transaction, and 2) transactions involving contracts that can be influenced by multiple entities (e.g. covenants). (See [Recommendation of `SIGHASH_UTXOS` for Multi-Entity Transactions](#recommendation-of-sighashutxos-for-multi-entity-transactions).)

### Double Spend Proof Support

Transactions employing `SIGHASH_UTXOS` and transactions spending outputs containing tokens are not protected by either of the [beta specifications for Double Spend Proofs (DSP)](https://documentation.cash/protocol/network/messages/dsproof-beta). Support for these features are left to future proposals.

### CashAddress Token Support

Two new [`CashAddress` types](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/cashaddr.md#version-byte) are specified to indicate support for accepting tokens:

| Type Bits      | Meaning           |
| -------------- | ----------------- |
| `2` (`0b0010`) | Token-Aware P2PKH |
| `3` (`0b0011`) | Token-Aware P2SH  |

**Token-aware wallet software** – wallet software which supports management of tokens – should use these CashAddress version byte values in newly created addresses.

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

Several measurements of fungible token supply are standardized for wider ecosystem compatibility. (See [Specification of Token Supply Definitions](#specification-of-token-supply-definitions).)

By design, [Genesis Supply](#genesis-supply), [Reserved Supply](#reserved-supply), [Circulating Supply](#circulating-supply), and [Total Supply](#total-supply) of any fungible token category will not exceed `9223372036854775807` (the maximum VM number).

#### Genesis Supply

A token category's **genesis supply** is an **immutable, easily-computed, maximum possible supply**, known since the token category's genesis transaction. It overestimates total supply if any amount of tokens have been destroyed since the genesis transaction.

The genesis supply of a fungible token category can be computed by parsing the outputs of the category's genesis transaction and summing the `amount` of fungible tokens matching the category's ID.

#### Total Supply

A token category's **total supply** is the sum – at a particular moment in time – of **tokens which are either in circulation or may enter circulation in the future**. A token category's total supply is always less than or equal to its genesis supply.

The total supply of a fungible token category can be computed by retrieving all UTXOs which contain token prefixes matching the category ID, removing provably-destroyed outputs (spent to `OP_RETURN` outputs), and summing the remaining `amount`s.

Software implementations should emphasize total supply in user interfaces for token categories which do not meet the requirements for emphasizing [circulating supply](#circulating-supply).

#### Reserved Supply

A token category's **reserved supply** is the sum – at a particular moment in time – of tokens held in reserve by the issuing entity. **This is the portion of the supply which the issuer represents as "not in circulation".**

The reserved supply of a fungible token category can be computed by retrieving all UTXOs which contain token prefixes matching the category ID, removing provably-destroyed outputs (spent to `OP_RETURN` outputs), and summing the `amount`s held in prefixes which have either the `minting` or `mutable` capability.

#### Circulating Supply

A token category's **circulating supply** is the sum – at a particular moment in time – of tokens not held in reserve by the issuing entity. **This is the portion of the supply which the issuer represents as "in circulation".**

The **circulating supply** of a fungible token category can be computed by subtracting the [reserved supply](#reserved-supply) from the [total supply](#total-supply).

Software implementations might choose to emphasize circulating supply (rather than total supply) in user interfaces for token categories which:

- are issued by an entity trusted by the user, or
- are issued by a covenant (of a construction known to the verifier) for which token issuance is limited (via a strategy trusted by the user).

## Usage Examples

The following examples outline high-level constructions made possible by this proposal. The term **covenant** is used to describe a Bitcoin Cash contract that inspects the transaction that spends it, enforcing constraints – e.g. a contract that requires user to send an equal number of satoshis back to the same contract (such that the contract maintains its own "balance").

### Identity Tokens

Prior to this proposal, Bitcoin Cash contracts were limited to interacting with static identities, usually defined by one or more public keys. A contract might allow a particular identity to spend funds under certain conditions (e.g. after a fixed delay), but if an identity needs to update its public key(s), the migration would require that the contract be re-created and the funds moved.

This proposal enables contracts to instead interact with **identity tokens** – non-fungible tokens that prove control of a represented identity. Identity tokens can be moved independently of the contracts that verify them, allowing greater flexibility and control over the identities within contracts, e.g. a user may upgrade to a two-factor (multisig) wallet, a company may rotate keys after a security breach, or an organization may add or remove individuals with signing authority. With identity tokens, contracts can be designed to allow identities to perform these updates without re-creating the contract, reducing the cost and complexity of coordinating with other contract participants.

### Covenant-Tracking Identity Tokens

A covenant can be associated with a **"tracking" identity token**, requiring that spends always re-associate the identity token with the covenant.

Beyond simplifying logic for clients to safely locate and interact with the covenant, such a tracking token offers an impersonation-proof strategy for other contracts to authenticate a particular covenant instance. This primitive enables covenants to design **public interfaces**, paths of operation intended for other contracts (which may themselves be designed and deployed after the creation of the original covenant).

Because token category IDs can be known prior to their creation, it is straightforward to create ecosystems of contracts that are mutually-aware of each other's tracking token category ID(s).

Notably, tracking tokens also allow for a significant contract-size and application-layer optimization: a covenant's internal state can be written to its tracking token's `commitment`, **allowing the locking bytecode of covenants to remain unchanged across transactions**.

### Depository Child Covenants

Given the existence of [covenant-tracking identity tokens](#covenant-tracking-identity-tokens), it is trivial to develop **depository child covenants** – covenants that hold a non-fungible token and/or an amount of fungible tokens on behalf of a parent covenant. By requiring that the depository covenant be spent only in transactions with the parent covenant, such depository covenants can allow a parent covenant to hold and manipulate token portfolios with an unlimited quantity of fungible or non-fungible tokens (despite the limitation to [one prefix codepoint per output](#one-prefix-codepoint-per-output)).

### Voting with Fungible Tokens

This proposal allows decentralized organization-coordinating covenants to hold votes using fungible tokens (without pausing the ability to transfer, divide, or merge fungible voting token outputs during the voting period).

In short, covenants can migrate to a new token category over a voting period – shareholders trade in amounts of "pre-vote" shares, receiving "post-vote" shares, and incrementing their chosen result by the `amount` of share-votes cast.

This construction reveals additional consensus strategies for decentralized organizations:

- **Vote-dependent, post-vote token categories** – covenants that rely on absolute consensus – like certain sidechain bridges (which, e.g. must come to a consensus on an aggregate withdrawal transaction per withdrawal period and then penalize or confiscate dishonest shares) – can issue different categories of post-vote tokens based on the vote cast. This allows for transfer and trading of post-vote tokens even before the voting period ends. When different voting outcomes impact the value of voting tokens (after the voting period ends), such differences will **immediately appear in market prices of post-vote tokens**. This observation presents further consensus strategies employing hedging, prediction markets, synthetic assets, etc.
- **Covenant spin-offs** – in cases where a covenant-based organization plans a spin-off (e.g. a significant sidechain fork), covenant participants can be allowed to select between receiving shares in the new covenant or receiving an alternative compensation (e.g. a one-time BCH payout or additional shares in the non-forking covenant).

To support simultaneous shareholder voting on multiple questions or to avoid the need for a token category ID migration, votes can be performed by submitting shares to trustless covenants implementing other strategies:

- **voting receipts** – rather than issuing a post-vote fungible token, vote-accepting covenants can accept fungible tokens of the original category as deposits, issuing non-fungible depository "receipt" tokens that entitle the holder to reclaim the deposited amount of original fungible tokens after the voting period ends. Voting receipts can also retain information about the vote cast such that the vote-accepting covenant can allow voters to "uncast" their votes and immediately withdraw their deposited tokens (e.g. to divest).
- **single-use voting tokens** – likewise, vote-accepting covenants can also issue multiple batches of single-use tokens in exchange for deposits of the original category. Each set of single-use tokens can be required by different vote-accepting covenants, and redeeming the deposited tokens of the original category may require either a depository receipt (e.g. after all voting periods end) or an equal amount of each issued single-use voting token (in uses cases where votes may be uncast before the voting period ends).

Finally, note that the above discussion focuses on coordinating on-chain voting for on-chain entities; off-chain entities can always conduct votes using off-chain coordination strategies, e.g. by accepting votes via [Bitauth signature](https://github.com/bitauth/bitauth-cli) from any output holding fungible tokens as of a previously-announced block height or [Median Time Past](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki).

#### Sealed Voting

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

### Multithreaded Covenants

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

### Multi-Covenant, Decentralized Applications

A technical demonstration has been prepared for this proposal: [Joint-Execution Decentralized Exchange (Jedex)](https://github.com/bitjson/jedex) is a multi-covenant, decentralized exchange design that supports trading between a particular fungible token and Bitcoin Cash. A full [list of demonstrated concepts](https://github.com/bitjson/jedex#demonstrated-concepts), the [application-specific token API](https://github.com/bitjson/jedex#jedex-token-api), and the application's [available transaction types](https://github.com/bitjson/jedex#transaction-flow) are each documented.

## Rationale

This section documents design decisions made in this specification.

### Incompatibility of Token Fungibility and Token Commitments

Advanced BCH contract use cases require strategies for transferring authenticated commitments – messages attesting to ownership, authorization, credit, debt, or other contract state – from one contract to another (a motivation behind [PMv3](https://github.com/bitjson/pmv3)). These use cases often conflicted with previous, fungibility-focused token proposals (see [Colored Coins](#colored-coins)).

One key insight which precipitated this proposal's bifurcated fungible/non-fungible approach is: **token fungibility and token commitments are conceptually incompatible**.

Fungible tokens are (by definition) indistinguishable from one another. Fungible token systems must allow amounts of tokens to be freely divided and re-merged without tracking the precise flow of individual token units. Conversely, nonfungible tokens (as defined by this proposal) are most useful to contracts because they offer a strategy for issuing tamper-proof messages that can be read and acted upon by other contracts.

Any token standard that attempts to combine these primitives must contend with their conceptual incompatibility – "fungible" tokens with commitments are not strictly fungible (e.g. some covenants could reject certain commitments, so wallet software must "assay" quantities of such tokens) and must have either implicit or user-defined policies for splitting and merging commitments (increasing protocol complexity and impeding standardization).

By clearly separating the fungible and non-fungible use cases, this specification is able to reduce each to a more fundamental, VM-compatible primitive. Rather than exhaustively specifying minting, transfer, or destruction "policies" at the protocol level – or creating another subsystem in which such policies are user-defined – all such policies can be specified using the existing Bitcoin Cash VM bytecode.

### Selection of Commitment Types

Commitments lift the management of data to the transaction level, allowing contracts to trust that the committed data is authentic (a commitment cannot be counterfeited). The [byte-string commitment type](#byte-string-commitments) ("non-fungible tokens") enforces only the authenticity of a particular byte string, but the more specialized [numeric commitment type](#numeric-commitments) ("fungible tokens") enables additional state management at the transaction level: the `amount` held in a numeric commitment can be freely divided among additional outputs or merged into fewer outputs. Because this behavior is enforced by the protocol, contracts can offload (to the protocol) logic that would otherwise be required to emulate this splitting/merging functionality.

Specialized commitment types could be designed for other VM data types – booleans, bitfields, hashes, signatures, and public keys, but these additional specialized commitment types have been excluded from this proposal for multiple reasons.

#### Boolean Commitment Type

A specialized **boolean commitment type** offers little value because boolean values cannot be combined non-destructively: once "merged", the component boolean values cannot be extracted from a merged boolean value, so "boolean commitment merging" is equivalent to burning of the merged commitments (leaving only a single `true` or `false` commitment in their place). As such, byte-string commitments are already optimal for the representation of boolean values.

#### Bitfield Commitment Type

A specialized **bitfield commitment type** could offer additional value beyond byte-string commitments because merging and recombination of bitfields is potentially non-destructive: bits set to `1` in the bitfield can be preserved within a transaction, merging any bitfields of the same category to hold `1` bits at multiple bitfield offsets within the same output. While this could reduce the size of some contracts that employ bitfields for state management, this reduction would be achieved at the cost of protocol and wallet implementation complexity:

- Transaction validation would be required to count the instances of each set bit in input bitfields and verify that outputs do not set more instances at each offset, and
- Wallet implementations would be required to carefully manage bitfields to avoid destroying set bits (by e.g. accidentally merging two bitfields that each set a bit at the same offset).

Finally, many other uses for "bitfield commitments" are not fully compatible with this simple split/merge behavior, requiring complex contract logic or byte-string commitments anyways. For example, [multithreaded covenants](#multithreaded-covenants) can represent a thread's identity using a bitfield such that [one bit is assigned to each thread](https://github.com/bitjson/jedex#demonstrated-concepts). These bitfield commitments must be carefully controlled by covenants to ensure that thread IDs cannot be moved or changed unexpectedly, so in practice, a specialized bitfield commitment type could only reduce the size of these covenants by a few bytes per transaction vs. equivalent covenants designed to use byte-string commitments. Because these slight efficiency gains apply only to a small subset of contracts, the required increase in protocol and wallet implementation complexity for a specialized bitfield commitment type cannot be justified.

#### Hash Commitment Types

Two specialized **hash commitment types** could be designed to allow commitments containing hashes to be merged into a single output and later separated; the design requirements of each type depends on the secrecy of the hash preimage.

##### Secret Hash Commitment Type

Many decentralized oracle use cases require participants to commit to secret values prior to a reveal period (e.g. [sealed voting](#sealed-voting)). For a hash commitment type to be useful in these applications, the type must support merging and splitting of commitments without revealing the hashed contents.

This **secret hash commitment type** would require a protocol-standardized commitment data structure to/from which sets of hashed values can be added or removed (e.g. a standard Merkle tree), and efficient utilization within contracts would require new VM operations that allow the contract to verify that one or more hashes are included or excluded from the data structure (e.g. [`OP_MERKLEBRANCHVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0116.mediawiki)). Additionally, many use cases also require nonsecret/validated metadata (e.g. a block height, block time, a count of shares cast, etc.) in addition to the hash of the secret preimage, so a secret hash commitment type would need to either 1) support associating non-secret data with each hash, or 2) defer to byte-string commitments for such use cases.

In practice, contracts can already define equivalent, application-specific data structures and validation strategies using VM bytecode and byte-string commitments; because these efficiency gains apply only to a small subset of contracts, the required increase in protocol and wallet implementation complexity for a specialized secret hash commitment type cannot be justified.

##### Nonsecret Hash Commitment Type

A specialized **nonsecret hash commitment type** could be designed to support use cases that do not require hashed data to remain secret during merging and splitting of commitments. Users could present structured data to some contract that verifies and/or records (to its internal state) the raw data before issuing a token committing to the hash of the data; the token could be presented later to a supporting contract to demonstrate that the raw data was previously verified or recorded by the issuing contract. This nonsecret hash commitment type would support only a subset of the contracts that might be made more efficient by a [secret hash commitment type](#secret-hash-commitment-type), but this tradeoff could allow for a simpler commitment data structure (e.g. canonically-ordered, length-prefixed preimages) and simpler contract verification strategies.

As with a secret hash commitment type, efficient utilization of a nonsecret hash commitment type within contracts would require new VM operations that allow the contract to verify that one or more hashes are included or excluded from the commitment data structure (e.g. incremental hashing operations). Because byte-string commitments and the existing VM bytecode system can already be used to define equivalent, application-specific data structures and validation strategies, the required increase in protocol and wallet implementation complexity for a specialized nonsecret hash commitment type cannot be justified.

#### Public Key Commitment Type

A specialized **public key commitment type** could be designed to support use cases in which a contract issues commitments containing public keys, allowing the committed public keys to be aggregated (and possibly redivided among outputs). Such a type would require the protocol-standardization of a particular public key aggregation scheme and (likely) new VM operations to fully-enable usage of the scheme within contracts.

Even with these protocol changes, a practical use case for such a commitment type has not been identified. Most contracts issuing public keys in commitments as an authorization strategy – allowing the "public key token" to be provided with a signature from the public key – could be more efficiently implemented by simply accepting the token itself as proof of authorization; providing an additional signature by the committed public key would be extraneous. In cases where the additional signature is useful, it is not clear that support for aggregating and/or redividing committed public keys would allow contracts to offload any logic to the protocol: if a contract issues commitments for multiple public keys to different entities, each entity gains nothing from aggregating their public key with other entities (effectively burning their ability to individually sign for the committed public key); likewise, if division of aggregated public keys is supported, contracts have little purpose in issuing commitments of aggregated public keys, as the token holder could always divide the aggregated public key and use the token with only one of the component public keys. As such, the required increase in protocol and wallet implementation complexity for a specialized public key commitment type cannot be justified.

#### Signature Commitment Type

As with public keys, a specialized **signature commitment type** could be designed to support signature aggregation, but such a commitment type offers no additional value in contract applications. Signatures are self-authenticating data – they can be freely copied and validated directly from unlocking bytecode; any signature commitment could be replaced by an equivalent empty commitment, and the signature instead provided in the unlocking bytecode, without impacting security. As such, the required increase in protocol and wallet implementation complexity for a specialized public key commitment type cannot be justified.

### Shared Codepoint for All Tokens

Though fungible and non-fungible tokens are independent primitives, this specification defines the same `PREFIX_TOKEN` codepoint for encoding both token types.

Alternatively, additional codepoints could be reserved to save a byte for outputs that encode only tokens of one or the other type (e.g. "`PREFIX_FUNGIBLE`" and "`PREFIX_NONFUNGIBLE`"). However, this change would require matching of the additional prefixes during decoding – increasing decoding complexity in the common case (if tokens are not present on the majority of outputs) – and the potential savings are insufficient to justify expending additional codepoints.

### Behavior of Minting and Mutable Tokens

This specification includes support for adding two different "capabilities" to non-fungible tokens: `minting` and `mutable`.

Minting tokens allow the holder to create new non-fungible tokens that share the same category ID as the minting token. By implementing this minting capability for a category (rather than requiring all category tokens to be minted in a single transaction), this specification enables use cases that require both a stable category ID and long-running issuance. This is critical for both primarily-off-chain applications of non-fungible tokens (where issuers commonly require both stable identifiers and an issuer-provided commitment) and for most covenant use cases (where covenants must be able to create new commitments in the course of operation).

By implementing category-minting control as a token, minting policies can be defined using complex contracts (e.g. multisig vaults with time-based fallbacks) or even covenants (e.g. to enforce uniqueness of commitments within the category, to issue receipts or other bearer instruments to covenant users, etc.). Notably, this specification even allows minting tokens to create additional minting tokens; this is valuable for [many specialized covenants](#multithreaded-covenants) that require the ability to delegate token-creation authority to other covenants.

Mutable tokens allow the holder to create **only one new token** (i.e. "modify" the mutable token's commitment) that may again have the mutable capability.

This is a particularly critical use case for covenants, as it enables covenants to modify the commitment in a single NFT (e.g. [tracking token](#covenant-tracking-identity-tokens)) without exhaustively validating that the interaction did not unexpectedly mint new tokens (allowing the spender to impersonate the covenant). While exhaustive validation could be made efficient with new VM opcodes (like [loops](https://github.com/bitjson/bch-loops)), such validation may commonly conflict across covenants, preventing them from being used in the same transaction. As such, this proposal considers the `mutable` capability to be essential for [cross-covenant interfaces](#cross-contract-interfaces).

### Exclusion of Cloneable Capability

This proposal could specify a "cloneable" capability that would allow the spender to create new NFTs with the same commitment as the cloneable NFT. Such a capability could simplify use cases where a single commitment must be consumed by multiple contracts. However, more byte-efficient strategies are already available for implementing this behavior (e.g. ["certification tokens"](https://bitcoincashresearch.org/t/chip-2022-02-cashtokens-token-primitives-for-bitcoin-cash/725/21)), and a "cloneable" capability would both increase protocol complexity and encourage inefficient application designs that duplicate commitment data in the UTXO set.

### Avoiding Proof-of-Work for Token Data Compression

Previous token proposals require token creators to retry hashing preimages until the resulting token category ID matches required patterns. This strategy enables additional bits of information to be packed into the category ID.

In practice, such proof-of-work strategies unnecessarily complicate covenant-managed token creation. To create [ecosystems of contracts that are mutually-aware of each other contract](#covenant-tracking-identity-tokens), it is valuable to be able to predict the category ID of a token that has not yet been created; hashing strategies often preclude such planning.

Finally, at a software level, rapid retrying of hash preimages is expensive to implement and audit. Wallet software must recognize which fields may be modified, and some logic must select the parameters of each attempt. This additional flexibility presents a large surface area for exfiltration of key material (i.e. hiding parts of the private key in various modifiable structures). Signing standards for mitigating this risk may be expensive to specify and implement.

### Use of Transaction IDs as Token Category IDs

In defining token category IDs, this proposal makes a tradeoff: new token categories can only be created using outpoints with an index of `0` (UTXOs that were the 0th output of a transaction), and in exchange, **existing transaction indexes can be used to locate token genesis transactions** (token category creation transactions).

This tradeoff significantly simplifies implementations in node and indexing software, and it **allows wallet software to easily query and verify both genesis transaction information and later token history**. This can be useful for verifying token supply, inspecting data included in any genesis transaction `OP_RETURN` outputs, or performing other protocols (e.g. inspecting [Bitauth](https://github.com/bitauth/bitauth-cli) metadata).

Additionally, the one-category-per-transaction requirement is trivial to meet: token-creating wallet software can provide new category IDs using single-output (zero-confirmation) intermediate transactions; contracts that oversee token creation can require users to provide a sufficient number of category IDs (via such intermediate transactions) and need only verify that they have been provided (as the VM would reject invalid token category IDs prior to contract evaluation).

Finally, If real-world usage demonstrates demand, a future upgrade could enable token category creation with non-zero outpoint indexes by, e.g. allowing outputs with new token categories to use a hash of the full outpoint (both hash and index) in the corresponding input. (Such hashes must be 32 bytes to prevent birthday attacks.)

### One Token Prefix Per Output

Many use cases rely on the ability for one transaction output to effectively control multiple groups/types of tokens, e.g. managing multiple non-fungible tokens or trading between multiple categories of fungible tokens. This specification allows only one token prefix per output (and within it, only one NFT) for several reasons:

- **Simplified VM API** – because only one category of tokens can exist at any UTXO or output, the "token API" exposed via token inspection operations is significantly simplified: each operation requires only one index, and contracts need not handle combinatorial cases where tokens are placed at unexpected "sub-indexes" (in addition to validation across standard UTXO/output indexes). This reduces contract complexity and transaction sizes in the common case where contracts need only interact with one or two token categories.
- **Simplified data model** – limiting the data model to a single token prefix allows node implementations and indexing systems to support tokens without introducing another many-to-many relationship between some token "sub-output" and the existing output type (the most scaling-critical data type). The single-prefix limitation also allows token information to be easily denormalized into fixed-size output data structures.

Importantly, it should be noted that alternative strategies exist for effectively locking multiple (groups of) tokens to a single output. For example, a decentralized order book for trading non-fungible tokens could hold a portfolio of tokens in many separate [depository covenants](#depository-child-covenants), contracts that require the parent covenant to participate in any transactions. The sub-covenant can likewise verify the authenticity of the parent using a ["tracking" non-fungible token](#covenant-tracking-identity-tokens) that moves along with the parent covenant. (Or in cases where the sub-covenant moves with its parent in every transaction, it can also simply authenticate the parent by outpoint index after comparing with its own outpoint transaction hash.)

This parent-child strategy offers contracts practically unlimited flexibility in holding portfolios of both fungible and non-fungible tokens, and it encourages more efficient contract design across the ecosystem: rather than leaving large lists of token prefixes in a single large output (duplicating the full list in each transaction), transactions can involve only outputs with the tokens needed for a particular code path. Because contract development tooling will already require multi-output behavior, this optimization step is more likely to become widespread.

### Including Capabilities in Token Category Inspection Operations

The token category inspection operations (`OP_*TOKENCATEGORY`) in this proposal push the **concatenation of both category and capability** (if present). While token capabilities could instead be inspected with individual `OP_*TOKENCAPABILITY` operations, the behavior specified in this proposal is valuable for contract efficiency and security.

First, the combined `OP_*TOKENCATEGORY` behavior reduces contract size in a common case: every covenant that handles tokens must regularly compare both the category and capability of tokens. With the separated `OP_*TOKENCATEGORY` behavior, this common case would require at least 3 additional bytes for each occurrence – `<index> OP_*TOKENCAPABILITY OP_CAT`, and commonly, 6 or more bytes: `<index> OP_UTXOTOKENCAPABILITY OP_CAT` and `<index> OP_OUTPUTTOKENCAPABILITY OP_CAT` (`<index>` may require multiple bytes).

There are generally two other cases to consider:

- **covenants that hold mutable tokens** – these covenants are also optimized by the combined `OP_*TOKENCATEGORY` behavior. Because their mutable token can only create a single new mutable token, they need only verify that the user's transaction doesn't steal that mutable token: `OP_INPUTINDEX OP_UTXOTOKENCATEGORY <index> OP_OUTPUTTOKENCATEGORY OP_EQUALVERIFY` (saving at least 4 bytes when compared to the separated approach: `OP_INPUTINDEX OP_UTXOTOKENCATEGORY OP_INPUTINDEX OP_UTXOTOKENCAPABILITY OP_CAT <index> OP_OUTPUTTOKENCATEGORY <index> OP_OUTPUTTOKENCAPABILITY OP_CAT OP_EQUALVERIFY`).
- **covenants that hold minting tokens** – because minting tokens allow for new tokens to be minted, these covenants must exhaustively verify all outputs to ensure the user has not unexpectedly minted new tokens. (For this reason, minting tokens are likely to be held in isolated, minting-token child covenants, allowing the parent covenant to use the safer `mutable` capability.) For most outputs (verifying the output contains no tokens), both behaviors require the same bytes – `<index> OP_OUTPUTTOKENCATEGORY OP_0 OP_EQUALVERIFY`. For expected token outputs, the combined behavior requires a number of bytes similar to the separated behavior, e.g.: `<index> OP_OUTPUTTOKENCATEGORY <32> OP_SPLIT <0> OP_EQUALVERIFY <depth> OP_PICK OP_EQUALVERIFY` (combined, where `depth` holds the 32-byte category set via `OP_INPUTINDEX OP_UTXOTOKENCATEGORY <32> OP_SPLIT OP_DROP`) vs. `OP_INPUTINDEX OP_UTXOTOKENCATEGORY <index> OP_OUTPUTTOKENCATEGORY OP_EQUALVERIFY OP_INPUTINDEX OP_UTXOTOKENCAPABILITY <index> OP_OUTPUTTOKENCAPABILITY` (separated).

Beyond efficiency, this combined behavior is also **critical for the general security of the covenant ecosystem**: it makes the most secure validation (verifying both category and capability) the "default" and cheaper in terms of bytes than more lenient validation (allowing for other token capabilities, e.g. during minting).

For example, assume this proposal specified the separated behavior: if an upgradable (e.g. by shareholder vote) covenant with a tracking token is created without any other token behavior, dependent contracts may be written to check that the user has also somehow interacted with the upgradable covenant (i.e. `<index> OP_UTXOTOKENCATEGORY <expected> OP_EQUALVERIFY`). If the upgradable covenant later begins to issue tokens for any reason, a vulnerability in the dependant contract is exposed: users issued a token by the upgradable covenant can now mislead the dependent contract into believing it is being spent in a transaction with the upgradeable contract (by spending their issued token with the dependent contract). Because the dependent contract did not include a defensive `<index> OP_UTXOTOKENCAPABILITY <mutable> OP_EQUALVERIFY` (either by omission or to reduce contract size), it became vulnerable after a "public interface change". If `OP_UTXOTOKENCATEGORY` instead uses the combined behavior (as specified by this proposal) this class of vulnerabilities is eliminated.

Finally, this proposal's combined behavior preserves two codepoints in the Bitcoin Cash VM instruction set.

### Support for Zero-Length Commitments

This proposal allows for non-fungible token to have commitments with a length of `0`. This behavior matches that of stack items inside the VM, and it improves the efficiency and security of many contract constructions.

For many token categories, zero-length commitments can be used to represent concepts like memberships, authorizations, coupons, receipts, certifications, and more. In these cases, the availability of zero-length commitments often saves several bytes per transaction vs. 1-byte commitments (1 byte per token output, 1 to 2 bytes per validation) or fungible tokens (which require prior minting and handling of reserves).

Support for zero-length commitments also eliminates a possible contract vulnerability: token categories that commit to a single VM number (e.g. a counter or total in a covenant’s mutable token) could require that the next transaction perform a particular arithmetic operation on the committed number. If the required result is 0 (an empty VM stack item), and zero-length commitments were not supported, the covenant could be rendered unspendable (and funds permanently frozen). Workarounds include prefixing the committed number – costing at least 1 byte per token output (the prefix) and 2-4 bytes per validation (`<prefix> OP_CAT`, `<1> OP_SPLIT`) – or padding the committed number – costing 2-7 bytes per token output (`<8> OP_NUM2BIN`), and 1 byte per validation (`OP_BIN2NUM`). Supporting zero-length commitments avoids the need for contract audits to identify this issue or increase transaction sizes with workarounds.

Notably, zero-length commitments introduce a potential ambiguity in the state exposed to contracts by `OP_UTXOTOKENCOMMITMENT` and `OP_OUTPUTTOKENCOMMITMENT`; these operations return VM Number `0` for outputs with either 1) no NFT or 2) an immutable NFT with a commitment of length `0`. However, this ambiguity does not impact contract security, and it rarely increases transaction sizes. Contracts can differentiate between the cases by requiring the output in question to contain no fungible tokens (`OP_*TOKENAMOUNT <0> OP_EQUALVERIFY`), and wallets that are designed to interact with such contracts can split UTXOs containing both fungible tokens and zero-length-commitment, immutable NFTs into separate UTXOs prior to performing actions that require a zero-length-commitment, immutable NFT from a token category for which fungible tokens also exist.

Alternatively, this ambiguity could also be addressed by modifying `OP_UTXOTOKENCATEGORY` and `OP_OUTPUTTOKENCATEGORY` to include a final byte for "no capability" (i.e. an immutable NFT) rather than returning only the 32-byte token category. However, such a change would [de-optimize many common cases](#including-capabilities-in-token-category-inspection-operations) – requiring 3 additional bytes for most token validation – while only improving the efficiency of a rare case (validation of zero-length, immutable NFTs from categories that include fungible tokens) for which many efficient alternatives exist. Even for contract systems in which the rare case is applicable, this would increase transaction sizes (saving 2-3 bytes for the rare case, but costing 3 bytes per category validation in the surrounding contract).

### Limitation of Non-Fungible Token Commitment Length

This specification limits the length of non-fungible token commitments to `40` bytes; this restrains excess growth of the UTXO set and reduces the resource requirements of typical wallets and indexers.

By committing to a hash, contracts can commit to an unlimited collection of data (e.g. using a merkle tree). For resistance to [birthday attacks](https://bitcoincashresearch.org/t/p2sh32-a-long-term-solution-for-80-bit-p2sh-collision-attacks/750), covenants should typically use `32` byte hashes. This proposal expands this minimum requirement by `8` bytes to include an additional (padded, maximum-length) VM number. This is particularly valuable for covenants that are optimized to use multiple types of commitment structures and must concisely indicate their current internal "mode" to other contracts (e.g. re-organizing an unbalanced merkle tree of contract state for efficiency when the covenant enters "voting" mode). This additional 8 bytes also provides adequate space for higher-level standards to specify prefixes for committed hashes (e.g. marking identifiers to content-addressable storage).

### Inclusion of Token-Aware CashAddresses

This proposal adds two new types to the existing CashAddress format: `Token-Aware P2PKH` (`2`) and `Token-Aware P2SH` (`3`). By clearly designating "token-aware" equivalents of the existing address types, users are protected from unknowingly sending tokens to addresses where they may be inaccessible to the intended recipient.

By design, token support in most software remains optional, and many simple wallets may never have the need to implement token support (eliminating the need for token management user interfaces, simplifying UTXO management, and reducing maintenance costs). By introducing new, opt-in, token-aware CashAddress types, token-unaware software can continue to signal its inability to receive tokens, avoiding [lost funds and poor user experiences](https://github.com/bitjson/cashtokens/issues/32#issuecomment-1188122202).

### Recommendation of `SIGHASH_UTXOS` for Multi-Entity Transactions

This proposal recommends that all wallets enable the `SIGHASH_UTXOS` signing serialization flag when creating signatures for multi-entity transactions, including both 1) transactions where signatures are collected from multiple keys and assembled into a single transaction, and 2) transactions involving contracts that can be influenced by multiple entities (e.g. covenants). This recommendation eliminates several classes of vulnerabilities affecting light wallets and improves the privacy of all users of multi-entity transactions.

`SIGHASH_UTXOS` eliminates a set of potential vulnerabilities in light wallets and offline signing devices including [unexpected fee burning](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-August/014843.html), [unexpected token burning, theft, or unexpected actions in decentralized applications](https://github.com/bitjson/cashtokens/issues/22#issuecomment-1179333487).

Alternatively, signers could download and verify the spent output of each transaction referenced by each input prior to signing vulnerable transactions, but this approach would require significantly higher bandwidth, memory, and/or storage requirements for many light wallet and offline signing devices, impeding support for interacting with tokens or decentralized applications.

While these potential vulnerabilities do not impact signers with a full view of the necessary data (e.g. wallets with integrated, fully-validating nodes), because a subset of light wallets must enable `SIGHASH_UTXOS` for security, enabling `SIGHASH_UTXOS` for all signers improves privacy by maximizing the anonymity set of the resulting transactions. As such, this proposal recommends use of `SIGHASH_UTXOS` for all multi-entity transactions, even for non-vulnerable wallets.

### Limitation of Fungible Token Supply

Token validation has the effect of limiting fungible token supply to the maximum VM number (`9223372036854775807`). This is important for both contract usage and the token application ecosystem.

Consider decentralized exchange covenants that support submission of new liquidity pools: if it is possible to create a fungible token output with an amount greater than the maximum VM number, there is a class of vulnerabilities by which 1) the decentralized exchange fails to validate total amounts of newly submitted assets, 2) the error-producing value becomes embedded in the the covenant, and 3) the amount invalidates important spending paths, i.e. a denial of service (possibly permanently locking all funds). Because practically all contracts that handle arbitrary fungible tokens would need to employ this type of validation, limiting amounts to `9223372036854775807` reduces the size and complexity of a wide range of contracts.

Across the wider token application ecosystem (node software, indexers, and wallet software), the limit also makes maximum token supply simpler to calculate and display in user interfaces. Arbitrary-precision arithmetic is not necessary, and total supply has a maximum digit length of `19`.

Additionally, while a cap of `9223372036854775807` is likely sufficient for most cases (e.g. supporting $92 quadrillion at a precision of 1 cent), it is trivial to design contracts that consider two different token categories to be interchangeable. So while each token category is limited, a single genesis transaction can create a practically unlimited number of token categories, each with the maximum supply. Contracts can be designed to consider any of a set of token categories to be completely fungible, allowing practically unlimited "virtual" token amounts (but without requiring the wider token ecosystem to perform arbitrary-precision arithmetic or display the larger amounts in generalized user interfaces).

Finally, the Bitcoin Cash currency itself is significantly less granular than this limit – the maximum satoshi value is less than `2^53`, while fungible token supply extends to `2^63`. An upgrade providing for greater divisibility of fungible tokens should likely occur as part of an upgrade that increases the divisibility of the Bitcoin Cash currency itself (A.K.A. a "fractional satoshis" upgrade).

### Specification of Token Supply Definitions

The supply of covenant-issued tokens will often be inherently analyzable using the [supply-calculation algorithms](#fungible-token-supply-definitions) included in this specification. For example, all covenants that use a [tracking token](#covenant-tracking-identity-tokens) and retain a supply of unissued tokens will have an easily-calculable [circulating supply](#circulating-supply) (without requiring wallets/indexers to understand the implementation details of the contract).

By explicitly including these supply-calculation algorithms in this specification, token issuers are more likely to be aware of this reality, choosing to issue tokens via a strategy that conforms to common user expectations (improving data availability and compatibility across the ecosystem).

## Prior Art & Alternatives

Prior art and alternative token systems are reviewed below.

### PMv3

The CashTokens CHIP builds on ideas behind [PMv3](https://github.com/bitjson/pmv3) (2021), a version 3 transaction format proposal for Bitcoin Cash. The CashToken primitives were identified and extracted from covenant applications designed for PMv3 transactions (see [PMv3-Based CashTokens](https://blog.bitjson.com/cashtokens-contract-validated-tokens-for-bitcoin-cash/)), and the PMv3 proposal has been [withdrawn](https://bitcoincashresearch.org/t/chip-2021-01-pmv3-version-3-transaction-format/265/55?u=bitjson) in favor of CashTokens.

### Bitauth

[Bitauth](https://github.com/bitauth/bitauth2017/blob/40e84a073be3e8689e6f85b31a5b6536a3524238/bips/0-bitauth.mediawikis) (2016) is an identity resolution and message authentication protocol which uses transactions to define and update the public claims and signing requirements of identities. CashTokens' [use of Transaction IDs as Token Category IDs](#use-of-transaction-ids-as-token-category-ids) is derived from Bitauth's identity resolution strategy, and the [transferability of CashTokens' minting and mutable capabilities](#behavior-of-minting-and-mutable-tokens) is derived from Bitauth's migration transactions (differing from earlier [colored coin](#colored-coins) proposals that associated capabilities with fixed public keys or contract hashes).

CashTokens offer some functionality similar to (and compatible with) Bitauth: Bitauth could be extended to allow the representation of identities with non-fungible tokens, reducing client validation requirements and preventing outdated wallets from inadvertently destroying Bitauth identities. While marking a particular identity with an NFT would negatively impact privacy (by reducing its anonymity set from all unspent outputs to only unspent outputs with compatible NFTs), the identity would also become usable with on-chain entities.

### Colored Coins

Strategies for representing and managing new types of assets, beyond the Bitcoin Cash currency, on the Bitcoin Cash blockchain are generally referred to as **colored coin** systems. Colored coin proposals appeared shortly after the initial release of Bitcoin, some as exclusively application-layer protocols and some including proposed consensus changes. Ordered by date proposed, these include: [Mastercoin](https://bitcointalk.org/index.php?topic=101197.0;all), [Order-Based Coloring](https://github.com/killerstorm/colored-coin-tools/blob/master/colors.md#order-based-coloring), [OP_CHECKCOLORVERIFY](#op_checkcolorverify), [Freimarkets](#freimarkets), [Open Assets](https://github.com/OpenAssets/open-assets-protocol/blob/master/specification.mediawiki), [EPOBC](https://github.com/chromaway/ngcccbase/wiki/EPOBC_simple), [Colu](https://github.com/Colored-Coins/Colored-Coins-Protocol-Specification), [Confidential Assets](#confidential-assets),
[OP_GROUP](#op_group), [Group Tokenization](#group-tokenization), [SLPv1](#simple-ledger-protocol-v1), and [Unforgeable Groups](#unforgeable-groups).

Previous colored coin proposals focus primarily on support for fungible tokens; proposals that make provisions for _non-fungible tokens_ implement them as a fungible token category with a supply of 1 (Freimarkets, Open Assets, EPOBC, Colu, Group Tokenization, SLP, Unforgeable Groups). This approach typically fails to enable [contract-issued commitments](#contract-issued-commitments) and offers limited token-related functionality to contracts.

CashTokens differs from previous colored coin proposals in that CashTokens identifies [byte-string commitments](#byte-string-commitments) ("non-fungible tokens") as the core primitive required for advanced decentralized application development, and CashTokens identifies [numeric commitments](#numeric-commitments) ("fungible tokens") as a valuable but [incompatible specialization of byte-string commitments](#incompatibility-of-token-fungibility-and-token-commitments). These primitives are natural extensions of the existing Bitcoin Cash contracting system, enabling more advanced contract designs without impacting transaction or block validation costs.

#### OP_CHECKCOLORVERIFY

[OP_CHECKCOLORVERIFY](https://web.archive.org/web/20220313162546/https://bitcointalk.org/index.php?topic=253385.0%3Ball) (2013) was a proposal to redefine `OP_NOP3` to verify that a transaction could not counterfeit the "coloring" of satoshis (the smallest unit of the network's native currency). Satoshis would acquire a color via a minting protocol, and future minting would be controlled by the locking bytecode hashed to derive the color's identifier.

OP_CHECKCOLORVERIFY was similar to CashTokens in its VM-focused approach to supporting fungible tokens. However, OP_CHECKCOLORVERIFY token units would have used satoshis, so the use of tokens would carry an unnecessary opportunity cost in lost monetary time value. Additionally, OP_CHECKCOLORVERIFY would not have enabled [cross-contract interfaces](#cross-contract-interfaces) or composable, [decentralized applications](#decentralized-applications).

#### Freimarkets

[Freimarkets](https://web.archive.org/web/20160305014417/http://freico.in/docs/freimarkets.pdf) ([2013](https://web.archive.org/web/20160228230306/https://bitcointalk.org/index.php?topic=278671.0;all)) was a set of proposed changes for Bitcoin-like currencies which would enable fungible tokens and many use cases of fungible tokens: token interest/demurrage, atomic token exchanges, on-chain order matching, on-chain auctions, on-chain options, collectable NFTs, and several off-chain token usage strategies. Freimarkets included a new transaction format, transaction expiration, IEEE 754-2008 decimal floating point output amounts, arbitrary precision arithmetic, output "asset tags" (a 20-byte hash of the genesis transaction which created the asset type), sub-transactions, an "authorizer" subsystem for authorizing or charging fees on token transactions, a "validation script" subsystem (run when the block including a transaction is verified), a system for sidechain-like private ledgers, and a variety of new opcodes (`OP_BLOCK_HEIGHT`, `OP_BLOCK_TIME`, `OP_DELEGATION_SEPARATOR`, `OP_QUANTITY`, `OP_OUTPUT_SPENT`, `OP_OUTPUT_SPENT_IN`, `OP_OUTPUT_EXISTS`, and `OP_OUTPUT_EXISTS_IN`).

While many use cases for CashTokens could have been implemented with minor modifications to Freimarkets, contract constructions under Freimarkets would have been less efficient in terms of transaction size (requiring asset tag preimage inspection). Additionally, the magnitude of proposed changes presented an obstacle to deployment and adoption, and several components exposed the network to new attack vectors that require further analysis (arithmetic changes, transaction expiration, "authorizer" subsystem, "validation script" subsystem, opcode exposure of global state).

#### Confidential Assets

[Confidential Assets](https://blockstream.com/bitcoin17-final41.pdf) (2017) is a proposed upgrade for bitcoin-like cryptocurrencies which blinds the amounts of all UTXOs and adds support for issuing and managing assets with blinded identifiers ("asset tags") and amounts. The Confidential Assets proposal requires a new transaction format (with increased transaction sizes), additional cryptographic algorithms not currently in use by the protocol, and blinding of on-chain UTXO information.

CashTokens differs from Confidential Assets in that CashTokens enables [cross-contract interfaces](#cross-contract-interfaces) and composable, [decentralized applications](#decentralized-applications); Confidential Assets focuses primarily on representation of off-chain assets and cannot support covenant applications without additional upgrades. Additionally, CashTokens maintains the unblinded public auditability of all on-chain assets, while Confidential Assets makes public auditability more complex and reliant on the intractability of the discrete logarithm problem.

#### OP_GROUP

[OP_GROUP](https://web.archive.org/web/20171016150552/https://medium.com/@g.andrew.stone/bitcoin-scripting-applications-representative-tokens-ece42de81285) (2017) was a proposal to redefine `OP_NOP4` to verify that a transaction could not counterfeit the "coloring" of satoshis. Satoshis would acquire a color via a minting protocol, and future minting would be controlled by the public key used to derive the color's identifier. CashTokens differs from OP_GROUP in the same ways that CashTokens differs from [OP_CHECKCOLORVERIFY](#op_checkcolorverify).

#### Group Tokenization

Group Tokenization ([2018](https://web.archive.org/web/20181119165124/https://docs.google.com/document/d/1X-yrqBJNj6oGPku49krZqTMGNNEWnUJBRFjX7fJXvTs/edit), [2021](https://web.archive.org/web/20220315200939/https://www.bitcoinunlimited.net/grouptokenization/groupbchspec)) is a set of proposals to support colored coins and extend the contract system of bitcoin-like cryptocurrencies. Group Tokenization expands on the earlier [`OP_GROUP` proposal](#op_group) by migrating from satoshi coloring to unlimited minting, and it introduces authorities, capabilities (mint, melt, baton, rescript, and subgroup), authority migration transactions, a new subsystem for "script templates"/covenants, "subgroups", and a new transaction format. It also outlines research directions for new opcodes: `OP_EXEC` (similar to [`OP_EVAL`](https://github.com/bitcoin/bips/blob/master/bip-0012.mediawiki)) and `OP_PUSH_TX_DATA`.

CashTokens differs from Group Tokenization in that CashTokens enables [cross-contract interfaces](#cross-contract-interfaces) and composable, [decentralized applications](#decentralized-applications). While many CashTokens use cases could be implemented with the addition of token inspection operations to Group Tokenization, contract constructions under Group Tokenization would be more complex and less efficient in terms of transaction size (due to lack of non-fungible token primitive, genesis transaction hashing requirement, usage of subgroups to emulate commitments, lack of mutable tokens, and unlimited fungible token minting). Additionally, the magnitude of proposed changes present an obstacle to deployment and adoption, and several components require further specification and analysis (script template subsystem, issuer control of melt capability, new transaction format).

#### Simple Ledger Protocol (v1)

[Simple Ledger Protocol (SLP) version 1](https://slp.dev/specs/slp-token-type-1/) (2018) is the most widespread application-layer token system on Bitcoin Cash. SLP uses strictly formatted data carrier outputs (`OP_RETURN` outputs) to associate tokens with particular transaction outputs. SLP also includes a non-fungible token standard ([SLP NFT1](https://slp.dev/specs/slp-nft-1/), 2019).

The SLP-related contents of SLP transactions are not validated by the network, so SLP clients require an SLP-compatible indexing server and significant client-side validation (though trust-minimized, light-wallet validation is possible).

CashTokens differs from SLP in that CashTokens are validated by the network, and the CashToken primitives enable [cross-contract interfaces](#cross-contract-interfaces) and composable, [decentralized applications](#decentralized-applications). Additionally, SLP offers a complete token standard, including extensive implementation recommendations, best practices, and tooling. By design, the CashTokens CHIP specifies only the upgrade required to enable the CashToken primitives on Bitcoin Cash. Future specifications can incorporate CashTokens into more complete, application-specific standards.

#### Unforgeable Groups

[Unforgeable Groups](https://gitlab.com/0353F40E/group-tokenization/-/blob/808d9eb36caa1a64be0aa616d743b845dad67c7e/CHIP-2021-02_Unforgeable_Groups_for_Bitcoin_Cash.md) is a colored coin proposal which was derived from the earlier [Group Tokenization](#group-tokenization) proposal. CashTokens uses the [output prefix codepoint](#token-encoding) strategy from Unforgeable Groups to ensure backwards-compatibility of token encoding (obviating the need for a new transaction format). At publication, CashTokens differed from [Unforgeable Groups v6](https://gitlab.com/0353F40E/group-tokenization/-/blob/808d9eb36caa1a64be0aa616d743b845dad67c7e/CHIP-2021-02_Unforgeable_Groups_for_Bitcoin_Cash.md) in that CashTokens enables [cross-contract interfaces](#cross-contract-interfaces) and composable, [decentralized applications](#decentralized-applications); Unforgeable Groups has since been updated to recommend the CashTokens primitives.

## Stakeholders & Statements

> With these primitives, Bitcoin Cash can support decentralized applications comparable to Ethereum contract functionality, while retaining Bitcoin Cash’s >1000x efficiency advantage in transaction and block validation. [...] Because CashTokens could now enable more efficient, user-friendly decentralized prediction markets than PMv3, I’m withdrawing the PMv3 CHIP. —[Jason Dreyzehner, PMv3 CHIP Author](https://bitcoincashresearch.org/t/chip-2021-01-pmv3-version-3-transaction-format/265/55?u=bitjson)

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
  - [Libauth](https://github/com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash.
    - [Pull Request #98](https://github.com/bitauth/libauth/pull/98)
      - [Token encoding](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/message/transaction-encoding.ts#L395-L435)
      - [Token decoding](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/message/transaction-encoding.ts#L209-L324)
      - [Token-aware validation](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/vm/instruction-sets/bch/2023/bch-2023-tokens.ts#L163-L297)
      - [Token inspection operations](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/vm/instruction-sets/bch/2023/bch-2023-tokens.ts#L355-L389)
      - [Token-aware signing serialization](https://github.com/bitauth/libauth/blob/9dfa6cc0b8710dedfe007b47bd018f5a47079df5/src/lib/vm/instruction-sets/common/signing-serialization.ts#L456-L474)
      - [Token-aware CashAddresses](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/address/cash-address.ts#L61-L82)
      - [Verifying vmb_tests](https://github.com/bitauth/libauth/blob/43914ca973e90dfc84b6173dcace7233f8c2e05c/src/lib/vmb-tests/bch-vmb-tests.spec.ts#L122-L152)

## Feedback & Reviews

- [CashTokens CHIP Issues](https://github.com/bitjson/cashtokens/issues)
- [`CHIP 2022-02 CashTokens` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2022-02-cashtokens-token-primitives-for-bitcoin-cash/725)

## Acknowledgements

Thank you to the following contributors for reviewing and contributing improvements to this proposal, providing feedback, and promoting consensus among stakeholders:
[Calin Culianu](https://github.com/cculianu), [bitcoincashautist](https://github.com/A60AB5450353F40E), [imaginary_username](https://gitlab.com/im_uname), [Andrew Groot](https://github.com/thesquaregroot), [Tom Zander](https://github.com/zander), [Mathieu Geukens](https://github.com/mr-zwets), [Richard Brady](https://github.com/rnbrady), [Marty Alcala](https://github.com/msalcala11), [John Nieri](https://gitlab.com/emergent-reasons), [Jonathan Silverblood](https://gitlab.com/monsterbitar), [Benjamin Scherrey](https://github.com/scherrey), [Rosco Kalis](https://github.com/rkalis), [Deyan Dimitrov](https://github.com/dikel).

## Changelog

This section summarizes the evolution of this document.

- **v2.2.0 – 2022-9-30** (current)
  - Compress token encoding using bitfield
  - Encode mutable capability as `0x01` and minting capability as `0x02`
  - Revert to limiting `commitment_length` by consensus (`40` bytes)
  - Revert `PREFIX_TOKEN` to unique codepoint (`0xef`)
  - Modify `OP_*TOKENCOMMITMENT` to push `0` for zero-length commitments
  - Extend BIP69 sorting algorithm to support tokens
  - Specify activation times
  - Expand test vectors
  - Note non-support of beta specs for double spend proofs
  - Improve rationale
- **v2.1.0 – 2022-6-30** ([`f8b500a0`](https://github.com/bitjson/cashtokens/blob/f8b500a051f82d42dbf9e9e890bc6cdc14592307/readme.md))
  - Expand motivation, benefits, rationale, prior art & alternatives
  - Simplify token encoding, update test vectors
  - Set `PREFIX_TOKEN` to `0xd0` and limit `commitment_length` using standardness
  - Specify handling of pre-activation token-forgery outputs
  - Specify token-aware signing serialization algorithm and `SIGHASH_UTXOS`
- **v2.0.1 – 2022-2-25** ([`fcb110c3`](https://github.com/bitjson/cashtokens/blob/fcb110c3309901886b2c7d3417568d8b13fb01b5/readme.md))
  - Expand rationale
  - Note impossibility of valid token outputs in coinbase transactions
- **v2.0.0 – 2022-2-22** ([`879c55ed`](https://github.com/bitjson/cashtokens/blob/879c55edd7e9cd6a2c2d50990d89e5cc7cb07394/readme.md))
  - Initial publication (versioning begins at v2 to differentiate from [CashTokens v1](https://blog.bitjson.com/cashtokens-contract-validated-tokens-for-bitcoin-cash/))

## Copyright

This document is placed in the public domain.
