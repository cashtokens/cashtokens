# CHIP-2022-02-CashTokens: Token Primitives for Bitcoin Cash

        Title: Token Primitives for Bitcoin Cash
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Specification Version: 2.0.1
        Initial Publication Date: 2022-02-22
        Latest Revision Date: 2022-02-25

## Summary

This proposal enables two new primitives on Bitcoin Cash: **fungible tokens** and **non-fungible tokens**.

### Terms

A **token** is an asset – distinct from the Bitcoin Cash currency – that can be created and transferred on the Bitcoin Cash network.

**Fungible tokens** are a token type in which individual units are undifferentiated – groups of fungible tokens can be freely divided and merged without tracking the identity of individual tokens (much like the Bitcoin Cash currency).

**Non-Fungible tokens (NFTs)** are a token type in which individual units cannot be merged or divided – each NFT contains a **commitment**, a short, binary message attested to by the issuer of the NFT.

## Deployment

Deployment of this specification is proposed for the May 2023 upgrade.

## Motivation

Bitcoin Cash contracts lack primitives for trustlessly issuing transferable, contract-verifiable messages, limiting coordination strategies available to multi-party contracts and [covenants](#usage-examples).

## Benefits

By enabling token primitives on Bitcoin Cash, this proposal offers several benefits.

### Cross-Contract Interfaces

Using **non-fungible tokens** (NFTs), contracts can **create messages which can be read by other contracts**. These messages are impersonation-proof: other contracts can safely read and act on the commitment, certain that it was produced by the claimed contract.

With contract interoperability, behavior can be broken into clusters of smaller, coordinating contracts, **reducing transaction sizes**. This interoperability further enables [covenants](#usage-examples) to communicate over **public interfaces**, allowing diverse ecosystems of compatible covenants to work together, even when developed and deployed separately.

Critically, this cross-contract interaction can be achieved within the "stateless" transaction model employed by Bitcoin Cash, rather than coordinating via shared global state (like that required of the Ethereum virtual machine). This allows Bitcoin Cash to support comparably contract functionality while retaining its [>1000x efficiency advantage](https://blog.bitjson.com/pmv3-build-decentralized-applications-on-bitcoin-cash/#stateless-network-stateful-covenants) in transaction and block validation.

### Decentralized Applications

Beyond enabling covenants to interoperate with other covenants, these token primitives allow for byte-efficient representations of complex internal state – supporting advanced, decentralized applications on Bitcoin Cash.

**Fungible tokens** are critical for covenants to represent on-chain assets – e.g. voting shares, utility tokens, collateralized loans, prediction market options, etc. – and implement [complex coordination tasks](#voting-with-fungible-tokens) – e.g. liquidity-pooling, auctions, voting, sidechain withdrawals, spin-offs, mergers, and more.

**Non-fungible tokens** are critical for coordinating activity trustlessly between multiple covenants, enabling [covenant-tracking tokens](#covenant-tracking-non-fungible-tokens), [depository child covenants](#depository-child-covenants), [multithreaded covenants](#multithreaded-covenants), and other constructions in which a particular covenant instance must be authenticated.

### Universal Token Primitives

By exposing basic, consensus-validated token primitives, this proposal supports the development of higher-level, interoperable token standards (e.g. [SLP](https://slp.dev/)). Token primitives can be held by any contract, wallets can easily verify the authenticity of a token or group of tokens, and tokens [cannot be inadvertently destroyed](#disallowing-implicit-destruction-of-immutable-tokens) by non-token-aware wallet software.

## Technical Specification

A new `PREFIX_TOKEN` codepoint is specified in the Bitcoin Cash virtual machine (VM) instruction set, and six new token inspection opcodes are introduced. Transaction validation is modified to support the new prefix codepoint, and CashAddress `types` with token support are specified.

### Output Prefix Codepoints

An **output prefix codepoint** is a codepoint in the VM instruction set which is encoded at index `0` of a transaction output's locking bytecode and removed from the locking bytecode before evaluation. A single output may only include one output prefix codepoint (see [One Prefix Codepoint Per Output](#one-prefix-codepoint-per-output)), and output prefix codepoints cause an error if encountered during VM evaluation (even within an unexecuted conditional branch).

**A prefix codepoint opts the containing output into additional validation when the output is created or spent**. This specification includes a single output prefix codepoint, `PREFIX_TOKEN`.

| Name           | Codepoint      | (Opcode-Equivalent) Description                                                                                                        |
| -------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `PREFIX_TOKEN` | `0xef` (`239`) | Error, even when found in an unexecuted conditional branch. (May occur before an output's locking bytecode to indicate locked tokens.) |

### Token Categories

Every token belongs to a **category** specified via an immutable, 32-byte **Token Category ID** assigned in the category's **genesis transaction** (the transaction in which the token category is initially created).

Token Category IDs must be selected from the creating transaction's outpoints filtered for indexes of `0` (i.e. where the spent UTXO was the first output in its creating transaction). As such, implementations can locate the genesis transaction of any category by identifying the transaction that spent the `0`th output of the transaction referenced by the category ID. (See [Use of Transaction IDs as Token Category IDs](#use-of-transaction-ids-as-token-category-ids).)

### `PREFIX_TOKEN`

`PREFIX_TOKEN` is defined at codepoint `0xef` (`239`) and is the first byte of an output's **token prefix**, a data structure that can encode a non-fungible token (NFT) and/or an amount of fungible tokens (FTs) of the same category:

```
PREFIX_TOKEN <category_id> <token_output_type> [ft_amount] [<nft_commitment_length><nft_commitment>]
```

1. `<category_id>` – After the `PREFIX_TOKEN` byte, a 32-byte **Token Category ID** is required.
2. `<token_output_type>` – Encodes whether the output carries a FT amount, an NFT, or both; Also encodes capability of the NFT, and whether the NFT carries a commitment:
   1. If bit 0 flag (`0x01`) is set – the annotation **encodes an amount of fungible tokens**. Read the next bytes as `ft_amount`.
   2. If bit 4 flag (`0x10`) is set - in additon to encoding an NFT, the annotation **encodes the NFT's commitment**. Read the next bytes as `nft_commitment_length` and then read the `nft_commitment`.
   3. Bits 5 and 6 of `token_output_type` will encode `nft_type` which is read as `nft_type = token_output_type & 0x60`
       1. `0x60` (the **`minting` capability**) – the encoded non-fungible token is considered a **minting token**.
       2. `0x40` (the **`mutable` capability**) – the encoded non-fungible token is considered a **mutable token**.
       3. `0x20` (no capability) – the encoded non-fungible token is considered an **immutable token**.
       4. `0x00` None - the output does not encode an NFT.
3. `ft_amount` – If `token_output_type & 0x01 != 0` then a **fungible token amount** (encoded as a `VarInt`) is required, with a minimum value of `0` (`0x00`) and a maximum value equal to the maximum VM number, `9223372036854775807` (`ffffffffffffffff7f`).
4. `nft_commitment_length` – If `token_output_type & 0x10 != 0` then the NFT's **commitment length** (encoded as a single-byte `VarInt`<sup>1</sup>) is required, with a minimum value of `1` (`0x01`) and maximum value of `40` (`0x28`). The NFT commitment will share consensus size limit with the locking script, while maximum value of 40 will be a network rule.
5. `nft_commitment` – If `token_output_type & 0x02 !=` then the NFT's **commitment** of `nft_commitment_length` size is required.

<details>

<summary>Notes</summary>

1. The **`VarInt` Format** is a variable-length, little-endian, positive integer format used to indicate the length of the following byte array in Bitcoin Cash P2P protocol message formats (present since the protocol's publication in 2008). For most of this range – `1` (`0x01`) to `40` (`0x28`) – the `VarInt` format is equivalent to the Number encoding used by the VM. (`VarInt` encodes `0` as `0x00`, while VM Numbers encode `0` as empty stack items.)

</details>

`PREFIX_TOKEN` must encode at least one token (non-fungible, fungible, or both) i.e. `token_output_type == 0x00` fails the transaction.  
The NFT commitment flag (`0x10`) may be set only if the output encodes an NFT i.e. `token_output_type & 0x70 == 0x10` fails the transaction.  
The commitment field is not required i.e. "null" commitment NFTs are allowed.  
If the NFT commitment field is used, it can not be "empty" i.e. `nft_commitment_length == 0` fails the transaction.  
Undefined bits are reserved i.e. `(token_output_type & 0x8E) != 0` fails the transaction.  
Fungible tokens of `ft_amount == 0` are allowed and distinct from outputs that don't encode a fungible token amount.

Implementations must ensure all token prefixes in newly created transaction outputs satisfy these encoding requirements during transaction validation.

<details>

<summary><strong><code>token_output_type</code> List of Allowed Values</strong></summary>


token_output_type | bit7 | bit6 | bit5 | bit4 | bit3 | bit2 | bit1 | bit0 | Note
-- | -- | -- | -- | -- | -- | -- | -- | -- | --
0x01 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 1 | FT, no NFT, no message
0x20 | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 0 | No FT, immutable NFT, no commitment
0x21 | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 1 | FT, immutable NFT, no commitment
0x30 | 0 | 0 | 1 | 1 | 0 | 0 | 0 | 0 | No FT, immutable NFT, carries a commitment
0x31 | 0 | 0 | 1 | 1 | 0 | 0 | 0 | 1 | FT, immutable NFT, carries a commitment
0x40 | 0 | 1 | 0 | 0 | 0 | 0 | 0 | 0 | No FT, mutable NFT, no commitment
0x41 | 0 | 1 | 0 | 0 | 0 | 0 | 0 | 1 | FT, mutable NFT, no commitment
0x50 | 0 | 1 | 0 | 1 | 0 | 0 | 0 | 0 | No FT, mutable NFT, carries a commitment
0x51 | 0 | 1 | 0 | 1 | 0 | 0 | 0 | 1 | FT, mutable NFT, carries a commitment
0x60 | 0 | 1 | 1 | 0 | 0 | 0 | 0 | 0 | No FT, mint NFT, no commitment
0x61 | 0 | 1 | 1 | 0 | 0 | 0 | 0 | 1 | FT, mint NFT, no commitment
0x70 | 0 | 1 | 1 | 1 | 0 | 0 | 0 | 0 | No FT, mint NFT, carries a commitment
0x71 | 0 | 1 | 1 | 1 | 0 | 0 | 0 | 1 | FT, mint NFT, carries a commitment

Values `0x00`, `0x10`, and `0x11` are disabled and will immediately fail the transaction.

All other values (those using any of the undefined bits, i.e. bit 7 and bits 1 through 3) are reserved and reading a reserved value will immediately fail the transaction.

</details>

<details>

<summary><strong>Output Parsing Algorithm</strong></summary>

  - set `prefix_token_length = 0`,
  - read the next 8 bytes as `satoshi_amount`,
  - read the next VarInt bytes as `locking_script_length`,
  - verify `locking_script_length <= MAX_LOCKING_SCRIPT_LENGTH + 43` _(verify the claimed length first, 43 is the allowance for FT of 8-byte amount because at this point it is indeterminate whether the output encodes a FT)_,
  - read the next byte as `locking_script[0]`,
  - if `locking_script[0] == PREFIX_TOKEN` then:
    - increment `prefix_token_length` by 1,
    - read the next 32 bytes as `category_id`, increment `prefix_token_length` by 32,
    - read the next byte as `token_output_type`, increment `prefix_token_length` by 1,
    - verify `token_output_type`:
      - if `token_output_type == 0x00` then fail,
      - if `token_output_type & 0x70 == 0x10` then fail,
      - if `token_output_type & 0x8e != 0` then fail,
    - if `(token_output_type & 0x01)` then:
      - read the next VarInt bytes as `ft_amount`, increment `prefix_token_length` by serialized size of the VarInt (1, 3, 5, or 9) ,
      - verify `ft_amount <= 2^63 - 1`,
    - if `(token_output_type & 0x10)` then:
      - read the next VarInt bytes as `nft_commitment_length` and increment `prefix_token_length` by serialized size of the VarInt (1, 3, 5, or 9),
      - verify `prefix_token_length + nft_commitment_length <= MAX_LOCKING_SCRIPT_LENGTH + 1`,
      - read the next `nft_commitment_length` bytes as `nft_commitment` and increment `prefix_token_length` by `nft_commitment_length`,
    - read the next `(locking_script_length - prefix_token_length)` bytes as `real_locking_script`
  - else:
    - read next `locking_script_length - 1` bytes into `locking_script`
    - set `real_locking_script = locking_script`
  - _set `prefix_token_length = 0`,_
  - _read the next 8 bytes as `satoshi_amount`..._

</details>

<details>

<summary><strong><code>PREFIX_TOKEN</code> Encoding Test Vectors</strong></summary>

The following test vectors demonstrate valid and invalid `PREFIX_TOKEN` encodings. The token category ID is `0xdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd` and commitments use repetitions of `0xcc`.

#### Valid `PREFIX_TOKEN` Prefix Encodings

| Description                                        | Encoded (Hex)                                                                                                                                                              |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| no NFT; 1 fungible                                 | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd0101`                                                                                                   |
| no NFT; 252 fungible                               | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd01fc`                                                                                                   |
| no NFT; 253 fungible                               | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd01fdfd00`                                                                                               |
| no NFT; 9223372036854775807 fungible               | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd01ffffffffffffffff7f`                                                                                   |
| null commitment, immutable NFT; no fungible                             | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd20`                                                                                                   |
| null commitment, immutable NFT; 1 fungible                             | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd2101`                                                                                                   |
| null commitment, immutable NFT; 253 fungible                           | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd21fdfd00`                                                                                               |
| null commitment, immutable NFT; 9223372036854775807 fungible           | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd21ffffffffffffffff7f`                                                                                   |
| 1-byte commitment, immutable NFT; 252 fungible                           | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd31fc01cc`                                                                                                 |
| 2-byte commitment, immutable NFT; 253 fungible                           | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd31fdfd0002cccc`                                                                                           |
| 10-byte commitment, immutable NFT; 65535 fungible                        | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd31fdffff0acccccccccccccccccccc`                                                                           |
| 40-byte commitment, immutable NFT; 65536 fungible                        | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd31fe0000010028cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc`           |
| 40-byte commitment, immutable NFT; no fungible                        | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd3028cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc`           |
| null commitment, mutable NFT; no fungible                    | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd40`                                                                                                 |
| null commitment, mutable NFT; 4294967295 fungible           | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd41feffffffff`                                                                                         |
| 1-byte commitment, mutable NFT; 4294967296 fungible           | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd41ff000000000100000001cc`                                                                               |
| 2-byte commitment, mutable NFT; 9223372036854775807 fungible  | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd41ffffffffffffffff7f02cccc`                                                                             |
| 10-byte commitment, mutable NFT; 1 fungible                   | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd41010acccccccccccccccccccc`                                                                             |
| 40-byte commitment, mutable NFT; 252 fungible                 | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd41fc28cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc`                 |
| 40-byte commitment, mutable NFT; no fungible                 | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd4028cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc`                 |
| null commitment, minting NFT; no fungible                    | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd60`                                                                                                 |
| null commitment, minting NFT; 253 fungible                  | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd61fdfd00`                                                                                             |
| 1-byte commitment, minting NFT; 65535 fungible                | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd71fdffff01cc`                                                                                           |
| 2-byte commitment, minting NFT; 65536 fungible                | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd71fe0000010002cccc`                                                                                     |
| 10-byte commitment, minting NFT; 4294967297 fungible          | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd71ff00000001000000010acccccccccccccccccccc`                                                             |
| 40-byte commitment, minting NFT; 9223372036854775807 fungible | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd71ffffffffffffffff7f28cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` |
| 40-byte commitment, minting NFT; no fungible | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd7128cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` |

#### Invalid `PREFIX_TOKEN` Prefix Encodings

| Reason                                                                                                    | Encoded (Hex)                                                                                                                                              |
| --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PREFIX_TOKEN` must encode at least one token                                                             | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd00`                                                                                   |
| `PREFIX_TOKEN` requires a token category ID                                                               | `ef`                                                                                                                                                       |
| Token category IDs must be 32 bytes                                                                       | `efdddddddd`                                                                                                                                               |
| Category must be followed by token information                                                            | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd`                                                                                       |
| NFT commitment length (41 bytes) must not be larger than 40 bytes                                             | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd31fc29cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` |
| NFT commitment length must be specified (mutable token)                                                       | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd51fc`                                                                                     |
| NFT commitment length must be specified (minting token)                                                       | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd71fc`                                                                                     |
| Not enough bytes remaining in locking bytecode to satisfy commitment length                               | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd31fc01`                                                                                     |
| Not enough bytes remaining in locking bytecode to satisfy commitment length                               | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd31fc02cc`                                                                                   |
| Token amount must be specified                                                                            | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd01`                                                                                     |
| Token amount (9223372036854775808) may not exceed 9223372036854775807                                     | `efdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd01ff0000000000000080`                                                                   |

</details>

### Token-Aware Transaction Validation

For any transaction to be valid, the [**token validation algorithm**](#token-validation-algorithm) must succeed.

This algorithm has the following effects:

1. **Universal Token Behavior**

   1. A single transaction can create multiple new token categories, and categories can contain both fungible and non-fungible tokens.
   2. Most tokens can not be implicitly destroyed by omission from a transaction's outputs – they must be explicitly destroyed by, for example, spending them to an `OP_RETURN` output. (Minting tokens and mutable tokens may be implicitly destroyed or modified. See [Behavior of Minting and Mutable Tokens](#behavior-of-minting-and-mutable-tokens).)
   3. Fungible and non-fungible tokens behave independently.
   4. A transaction output can contain both fungible tokens and a non-fungible token of the same category.

2. **Fungible Token Behavior**

   1. A transaction output can contain any amount of **fungible tokens** (of a single category).
   2. All fungible tokens of a particular category are created in the category's genesis transaction; their combined `amount` may not exceed `9223372036854775807`.
   3. A transaction can spend fungible tokens from any number of UTXOs to any number of outputs, so long as the sum of input and output `amount`s are equal (for each token category).

3. **Non-Fungible Token Behavior**

   1. A transaction output can contain zero or one **non-fungible token**.
   2. Non-fungible tokens (NFTs) of a particular category are created either in the category's genesis transaction or by later transactions which spend `minting` or `mutable` tokens of the same category.
   3. It is possible for multiple NFTs of the same category to carry the same commitment. (Uniqueness can be enforced by a covenant.)
   4. **Minting tokens** (NFTs with the `minting` capability) allow the spending transaction to create any number of new NFTs of the same category, each with any commitment and (optionally) the `minting` or `mutable` capability.
   5. Each **Mutable token** (NFTs with the `mutable` capability) allows the spending transaction to create one NFT of the same category, with any commitment and (optionally) the `mutable` capability.
   6. **Immutable tokens** (NFTs without a capability) cannot have their commitment modified when spent.

#### Token Validation Algorithm

Given the following **definitions**:

1. Reducing the set of UTXOs spent by the transaction:
   1. A key-value map of **`Input_Sums_By_Category`** (mapping category IDs to positive, 64-bit integers) is created by summing the `amount`s of each token category.
   2. A key-value map of **`Input_Mutable_Tokens_By_Category`** (mapping category IDs to positive integers) is created by summing the count of spent mutable tokens for each category.
   3. A de-duplicated list of **`Input_Minting_Categories`** is created including:
      1. The transaction hash of each outpoint with an outpoint index of `0` (i.e. the spent UTXO was the first output in its transaction), and
      2. the category ID of each spent minting token.
   4. A list of all **`Input_Immutable_Tokens`** (including duplicates) is created including each NFT which **does not** have a `minting` or `mutable` capability.
2. Reducing the set of outputs created by the transaction:
   1. A key-value map of **`Output_Sums_By_Category`** (mapping category IDs to positive, 64-bit integers) is created by summing the `amount`s of each token category.
   2. A key-value map of **`Output_Mutable_Tokens_By_Category`** (mapping category IDs to positive integers) is created by summing the count of spent mutable tokens for each category.
   3. A de-duplicated list of **`Output_Minting_Categories`** is created including the category ID of each created minting token.
   4. A list of all **`Output_Immutable_Tokens`** (including duplicates) is created including each NFT which **does not** have a `minting` or `mutable` capability.

Perform the following **validations**:

1. Each category in `Input_Sums_By_Category` must exist and have an equal sum in `Output_Sums_By_Category`.
2. Each category in `Output_Sums_By_Category` which doesn't exist in `Input_Sums_By_Category` must exist in `Minting_Category_IDs` and have an output sum no greater than `9223372036854775807` (the maximum VM number).
3. Each category in `Output_Mutable_Tokens_By_Category` must either:
   1. Exist and have a sum less than or equal to that category in `Input_Mutable_Tokens_By_Category`, or
   2. Exist in `Minting_Category_IDs`.
4. Each category in `Output_Minting_Categories` must exist in `Input_Minting_Categories`.
5. Match each token in `Input_Immutable_Tokens` to an exactly equal token in `Output_Immutable_Tokens` (comparing both category ID and commitment).
   1. `Input_Immutable_Tokens` must have no unmatched tokens.
   2. Derive a key-value map of **`Unmatched_Output_Tokens_By_Category`** which maps category IDs to a count of unmatched `Output_Immutable_Tokens`. Each category in `Unmatched_Output_Tokens_By_Category` must either:
      1. Exist in `Minting_Category_IDs`, or
      2. Have a count less than or equal to `Input_Mutable_Tokens_By_Category` minus `Output_Mutable_Tokens_By_Category` for that category.

Note: coinbase transactions have only one input with an outpoint index of `4294967295`, so they must never include a token prefix in any output.

#### Prefix Codepoint Standardness

Implementations must recognize otherwise-standard outputs with token prefixes (`PREFIX_TOKEN`) as **standard** (in `isStandard` validation).

### Token Inspection Operations

The following 6 operations pop the top item from the stack as an index (VM Number) and push a single result to the stack. If the consumed value is not a valid, minimally-encoded index for the operation, an error is produced.

| Name                       | Codepoint      | Description                                                                                                                                                                                                                                                                                                                    |
| -------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OP_UTXOTOKENCATEGORY`     | `0xce` (`206`) | Pop the top item from the stack as an input index (VM Number). Push the token category (with the 0xfe or 0xff capability byte appended, if present) of the Unspent Transaction Output (UTXO) spent by that input to the stack. If the UTXO includes no tokens, push a 0 (VM Number).                                           |
| `OP_UTXOTOKENCOMMITMENT`   | `0xcf` (`207`) | Pop the top item from the stack as an input index (VM Number). Push the token commitment of the Unspent Transaction Output (UTXO) spent by that input to the stack. If the UTXO includes a non-fungible token with a zero-byte commitment, push 0x00. If the UTXO does not include a non-fungible token, push a 0 (VM Number). |
| `OP_UTXOTOKENAMOUNT`       | `0xd0` (`208`) | Pop the top item from the stack as an input index (VM Number). Push the token amount of the Unspent Transaction Output (UTXO) spent by that input to the stack as a VM Number. If the UTXO includes no fungible tokens, push a 0 (VM Number).                                                                                  |
| `OP_OUTPUTTOKENCATEGORY`   | `0xd1` (`209`) | Pop the top item from the stack as an output index (VM Number). Push the token category (with the 0xfe or 0xff capability byte appended, if present) of the output at that index to the stack. If the output includes no tokens, push a 0 (VM Number).                                                                         |
| `OP_OUTPUTTOKENCOMMITMENT` | `0xd2` (`210`) | Pop the top item from the stack as an output index (VM Number). Push the token commitment of the output at that index to the stack. If the output includes a non-fungible token with a zero-byte commitment, push 0x00. If the output does not include a non-fungible token, push a 0 (VM Number).                             |
| `OP_OUTPUTTOKENAMOUNT`     | `0xd3` (`211`) | Pop the top item from the stack as an input index (VM Number). Push the token amount of the output at that index to the stack as a VM Number. If the output includes no fungible tokens, push a 0 (VM Number).                                                                                                                 |

<details>

<summary><strong>Token Inspection Operation Test Vectors</strong></summary>

The following test vectors demonstrate the expected result of each token inspection operation for a particular output. The `OP_UTXOTOKENCATEGORY` and `OP_OUTPUTTOKENCATEGORY` operations push the value in `Category`; the `OP_UTXOTOKENCOMMITMENT` and `OP_OUTPUTTOKENCOMMITMENT` operations push the value in `Commitment`; the `OP_UTXOTOKENAMOUNT` and `OP_OUTPUTTOKENAMOUNT` operations push the value in `Amount`. The token category ID is `0x1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d` and commitments use repetitions of `0xcc`.

| Description                                        | Category (Hex)                                                       | Commitment (Hex)                                                                   | Amount (Hex)         |
| -------------------------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------- |
| no NFT; 0 fungible                                 | (empty item)                                                         | (empty item)                                                                       | (empty item)         |
| no NFT; 1 fungible                                 | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | (empty item)                                                                       | `01`                 |
| no NFT; 252 fungible                               | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | (empty item)                                                                       | `fc`                 |
| no NFT; 253 fungible                               | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | (empty item)                                                                       | `fdfd00`             |
| no NFT; 9223372036854775807 fungible               | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | (empty item)                                                                       | `ffffffffffffffff7f` |
| 0-byte NFT; 0 fungible                             | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | `0x00`                                                                             | (empty item)         |
| 0-byte NFT; 1 fungible                             | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | `0x00`                                                                             | `01`                 |
| 0-byte NFT; 253 fungible                           | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | `0x00`                                                                             | `fdfd00`             |
| 0-byte NFT; 9223372036854775807 fungible           | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | `0x00`                                                                             | `ffffffffffffffff7f` |
| 1-byte NFT; 252 fungible                           | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | `cc`                                                                               | `fc`                 |
| 2-byte NFT; 253 fungible                           | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | `cccc`                                                                             | `fdfd00`             |
| 10-byte NFT; 65535 fungible                        | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | `cccccccccccccccccccc`                                                             | `fdffff`             |
| 40-byte NFT; 65536 fungible                        | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d`   | `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` | `fe00000100`         |
| 0-byte, mutable NFT; 0 fungible                    | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dfe` | `0x00`                                                                             | (empty item)         |
| 0-byte, mutable NFT; 4294967295 fungible           | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dfe` | `0x00`                                                                             | `feffffffff`         |
| 1-byte, mutable NFT; 4294967296 fungible           | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dfe` | `cc`                                                                               | `ff0000000001000000` |
| 2-byte, mutable NFT; 9223372036854775807 fungible  | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dfe` | `cccc`                                                                             | `ffffffffffffffff7f` |
| 10-byte, mutable NFT; 1 fungible                   | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dfe` | `cccccccccccccccccccc`                                                             | `01`                 |
| 40-byte, mutable NFT; 252 fungible                 | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dfe` | `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` | `fc`                 |
| 0-byte, minting NFT; 0 fungible                    | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dff` | `0x00`                                                                             | (empty item)         |
| 0-byte, minting NFT; 253 fungible                  | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dff` | `0x00`                                                                             | `fdfd00`             |
| 1-byte, minting NFT; 65535 fungible                | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dff` | `cc`                                                                               | `fdffff`             |
| 2-byte, minting NFT; 65536 fungible                | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dff` | `cccc`                                                                             | `fe00000100`         |
| 10-byte, minting NFT; 4294967297 fungible          | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dff` | `cccccccccccccccccccc`                                                             | `ff0100000001000000` |
| 40-byte, minting NFT; 9223372036854775807 fungible | `1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1d1dff` | `cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc` | `ffffffffffffffff7f` |

</details>

### Existing Introspection Operations

Note, this specification has **no impact on the behavior of [`OP_UTXOBYTECODE` or `OP_OUTPUTBYTECODE`](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Add-Native-Introspection-Opcodes.md)**. Both operations continue to return only the contents of the respective locking bytecode, excluding both the `VarInt`-encoded locking bytecode length and any output prefixes, if present.

### Interpretation of Signature Preimage Inspection

It is possible to design contracts which inefficiently inspect the encoding of tokens using **signature preimage inspection** – inspecting the contents of a preimage for which a signature passes both `OP_CHECKSIG(VERIFY)` and `OP_CHECKDATASIG(VERIFY)`.

This specification interprets all signature preimage inspection of tokens as **intentional**: these constructions are designed to succeed or fail based on the encoding of the signature preimage, and they can be used (by design) to test for 1) the availability of some types of proposed-but-not-activated upgrades, and/or 2) a contracts' presence on a fork of Bitcoin Cash. This notice codifies a network policy: the possible existence of these contracts will not preclude future upgrades from adding additional output prefix or transaction formats. (The security of a contract is the responsibility of the entity locking funds in that contract; funds can always be locked in insecure contracts, e.g. `OP_DROP OP_1`.)

Contract authors are advised to use [Token Inspection Operations](#token-inspection-operations) for all constructions intended to inspect the actual properties of tokens within a transaction.

### CashAddress Token Support

Two new [`CashAddress` types](https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/cashaddr.md#version-byte) are specified to indicate support for accepting tokens:

| Type Bits      | Meaning           | Version Byte Value  |
| -------------- | ----------------- | ------------------- |
| `2` (`0b0010`) | Token-Aware P2PKH | `16` (`0b00010000`) |
| `3` (`0b0011`) | Token-Aware P2SH  | `24` (`0b00011000`) |

**Token-aware wallet software** – wallet software which supports management of tokens – should use these CashAddress version byte values in newly created addresses.

Token-aware wallet software **must refuse to create token transactions including any CashAddresses without token support** (i.e. `P2PKH` – version `0`, and `P2SH` version `8`).

### Fungible Token Supply Definitions

Several measurements of fungible token supply are standardized for wider ecosystem compatibility. (See [Specification of Token Supply Definitions](#specification-of-token-supply-definitions).)

By design, [Genesis Supply](#genesis-supply), [Reserved Supply](#reserved-supply), [Circulating Supply](#circulating-supply), and [Total Supply](#total-supply) of any fungible token category will not exceed `9223372036854775807` (the maximum VM number).

#### Genesis Supply

A token category's **genesis supply** is an **immutable, easily-computed, maximum possible supply**, known since the token category's genesis transaction. It overestimates total supply if any amount of tokens have been destroyed since the genesis transaction.

The genesis supply of a fungible token category can be computed by parsing the outputs of the category's genesis transaction and summing the `amount` of fungible tokens matching the category's ID.

#### Total Supply

A token category's **total supply** is the sum – at a particular moment in time – of **tokens which are either in circulation or may enter circulation in the future**. A token category's total supply is always less than or equal to its genesis supply.

The total supply of a fungible token category can be computed by retrieving all UTXOs which contain token prefixes matching the category ID, removing provably-destroyed outputs (spent to `OP_RETURN`), and summing the remaining `amount`s.

Software implementations should emphasize total supply in user interfaces for token categories which do not meet the requirements for emphasizing [circulating supply](#circulating-supply).

#### Reserved Supply

A token category's **reserved supply** is the sum – at a particular moment in time – of tokens held in reserve by the issuing entity. **This is the portion of the supply which the issuer represents as "not in circulation".**

The reserved supply of a fungible token category can be computed by retrieving all UTXOs which contain token prefixes matching the category ID, removing provably-destroyed outputs (spent to `OP_RETURN`), and summing the `amount`s held in prefixes which have either the `minting` or `mutable` capability.

#### Circulating Supply

A token category's **circulating supply** is the sum – at a particular moment in time – of tokens not held in reserve by the issuing entity. **This is the portion of the supply which the issuer represents as "in circulation".**

The **circulating supply** of a fungible token category can be computed by subtracting the [reserved supply](#reserved-supply) from the [total supply](#total-supply).

Software implementations might choose to emphasize circulating supply (rather than total supply) in user interfaces for token categories which:

- are issued by an entity trusted by the user, or
- are issued by a covenant (of a construction known to the verifier) for which token issuance is limited (via a strategy trusted by the user).

## Usage Examples

The following examples outline high-level constructions made possible by this proposal. The term **covenant** is used to describe a Bitcoin Cash contract which inspects the transaction which spends it, enforcing constraints – e.g. a contract which requires that users send an equal number of satoshis back to the same contract (such that the contract maintains its own "balance").

### Covenant-Tracking, Non-Fungible Tokens

A covenant can be associated with a **"tracking" non-fungible token**, requiring that spends always re-associate the non-fungible token with the covenant.

Beyond simplifying logic for clients to safely locate and interact with the covenant, such a tracking token offers an impersonation-proof strategy for other contracts to authenticate a particular covenant instance. This primitive enables covenants to design **public interfaces**, paths of operation intended for other contracts (which may themselves be designed and deployed after the creation of the original covenant).

Because token category IDs can be known prior to their creation, it is straightforward to create ecosystems of contracts that are mutually-aware of each other's tracking token category ID(s).

Notably, tracking tokens also allow for a significant contract-size and application-layer optimization: a covenant's internal state can be written to its tracking token's `commitment`, **allowing the locking bytecode of covenants to remain unchanged across transactions**.

### Depository Child Covenants

Given the existence of [covenant-tracking, non-fungible tokens](#covenant-tracking-non-fungible-tokens), it is trivial to develop **depository child covenants** – covenants which hold a non-fungible token and/or an amount of fungible tokens on behalf of a parent covenant. By requiring that the depository covenant be spent only in transactions with the parent covenant, such depository covenants can allow a parent covenant to hold and manipulate token portfolios with an unlimited quantity of fungible or non-fungible tokens (despite the limitation to [one prefix codepoint per output](#one-prefix-codepoint-per-output)).

### Voting with Fungible Tokens

This proposal allows decentralized organization-coordinating covenants to hold votes using fungible tokens (without pausing the ability to transfer, divide, or merge fungible voting token outputs during the voting period).

In short, covenants can migrate to a new token category over a voting period – shareholders trade in amounts of "pre-vote" shares, receiving "post-vote" shares, and incrementing their chosen result by the `amount` of share-votes cast.

This construction reveals additional consensus strategies for decentralized organizations:

- **Vote-dependent, post-vote token categories** – covenants which rely on absolute consensus – like certain sidechain bridges (which, e.g. must come to a consensus on an aggregate withdrawal transaction per withdrawal period and then penalize or confiscate dishonest shares) – can issue different categories of post-vote tokens based on the vote cast. This allows for transfer and trading of post-vote tokens even before the voting period ends. When different voting outcomes impact the value of voting tokens (after the voting period ends), such differences will **immediately appear in market prices of post-vote tokens**. This observation presents further consensus strategies employing hedging, prediction markets, synthetic assets, etc.
- **Covenant spin-offs** – in cases where a covenant-based organization plans a spin-off (e.g. a significant sidechain fork), covenant participants can be allowed to select between receiving shares in the new covenant or receiving an alternative compensation (e.g. a one-time BCH payout or additional shares in the non-forking covenant).

#### Sealed Voting

**"Sealed" voting** – in which the contents of votes are unknown until after all votes are cast – is immediately possible with this proposal.

Voting begins with a "ballot box" merkle tree, containing at least as many empty leaves as outstanding shares. Voters submit a **sealed vote**: a message containing 1) the number of share-votes cast and 2) a hash of their vote concatenated with a [salt](<https://en.wikipedia.org/wiki/Salt_(cryptography)>). Sealed votes are submitted by replacing an empty leaf in the ballot box (as demonstrated in [CashTokens v0](https://gist.github.com/bitjson/a440232cebba8f0b2b6b9aa5db1fdb37)). Once the voting period has ended, each participant can reverse the process: prove the contents of sealed votes within the tree by submitting the preimage, then aggregating results in another part of the covenant's state.

This basic construction can be augmented for various use cases:

- **voting quorum** – requiring some minimum percentage of sealed votes before a voting period ends.
- **unsealing quorum** – requiring some minimum percentage of sealed votes to be unsealed before vote results can be considered final.
- **sealing deposits** – requiring voters to submit a deposit with sealed votes which can be retrieved by later unsealing the vote.
- **enforced vote secrecy** – allowing holders of either pre-vote or post-vote tokens to submit sealed "unsealing proofs", proving that another voter divulged their unsealed vote prior to the end of the voting period. Such proofs could reward the submitter at the expense of the prematurely-unsealing voter, frustrating attempts to coordinate malicious voting blocs. (A strategy [developed for Truthcoin](https://bitcoinhivemind.com/papers/truthcoin-whitepaper.pdf).)

(Note, secret ballot voting would require the introduction of additional VM opcodes.)

### Multithreaded Covenants

While most account-based contract models are globally-ordered by miners (e.g. Ethereum), the highly-parallel, UTXO model employed by Bitcoin Cash requires that **contract users determine transaction order**. This has significant scaling advantages – transactions can be validated in parallel, often before their confirming block is received, [enabling validation of 25,000 tx/second on modest hardware (as of 2020)](https://read.cash/@TomZ/scaling-bitcoin-cash-be8344a6). However, this model requires contract developers to account for transaction-order contention.

Transaction-order contention is of particular concern to **covenants**, contracts which require the spending transaction to match some pattern (e.g. returning the covenant's balance back to the same contract). Covenants typically offer a sort of "public interface", allowing multiple entities – or even the general public – to interact with a UTXO: depositing or withdrawing funds, trading tokens, casting votes, and more.

**Spend races** occur when multiple entities attempt to spend the same Bitcoin Cash UTXO. Spend races can degrade the user experience of interacting with covenants, requiring users to retry covenant transactions, and possibly preventing a user from interacting with the covenant at all.

To reduce disruptions from spend races, contracts must **carefully consider spend-race incentives**:

- To reduce frontrunning, covenants should allow actions to be submitted over time, **treating all submitted actions equally (regardless of submission time)** at some later moment.
- To disincentivize DOS attacks – e.g. where an attacker creates and rapidly broadcasts long chains of covenant transactions, spending each successive UTXO before other users can spend it in their own covenant transactions – covenants should **ensure covenant actions are authenticated and/or costly** (e.g. can only be taken once by each token holder, require some sort of fee or deposit, etc.).
- To resist censorship or orphaning of unconfirmed covenant transaction chains by malicious miners, covenants should **ensure important activity can occur over the course of many blocks**, and wallet software should maintain logic for recovering (possibly re-requesting authorization from the user) and retrying covenant transactions which are invalidated by malicious miners.

Beyond these basic strategies, this proposal enables another strategy: **multithreaded covenants** can offload logic to **"thread"** sub-covenants, allowing users to interact in parallel with multiple UTXOs. Threads aggregate state independently, allowing input or results to be "checked in" to the parent covenant in batches. Multithreaded covenants can identify authentic threads using [tracking non-fungible tokens](#covenant-tracking-non-fungible-tokens) (issued by the parent contract), and threads can authenticate both other threads and the parent covenant in the same way.

Thread design is application-specific, but valuable constructions include:

- **Lifetimes** – to avoid situations where the parent covenant is waiting on an overactive thread (e.g. a DOS attack), threads should often have a fixed lifetime – a decrementing internal counter which prevents the thread from accepting further transactions after reaching `0`. (And leaving a check-in with the parent covenant as the only valid spending method.)
- **Heartbeats** – a derivation of lifetimes for long-lived threads, heartbeats allow a thread's lifetime to be renewed to some fixed constant after a period (by validating locktime or sequence numbers). This guarantees occasional periods of inactivity during which a check-in can be performed.
- **Proof-of-work** – some threads may have use for rate limiting by proof-of-work, requiring users to submit preimages which hash to a value using some required prefix. (Note, for most applications, fees or minimum deposits offer more uniform rate limiting.)
- modified [**zero-confirmation escrows** (ZCEs)](https://github.com/bitjson/bch-zce) – and similar miner-enforced escrows can be employed by contracts to make abusive behavior more costly.

Given typical transaction propagation speed ([99% at 2 seconds](https://github.com/bitjson/bch-zce#transaction-conflict-monitoring)), multithreaded covenant applications with reasonable spend-race disincentives can expect **minimal contention between users so long as the available thread count exceeds `2` per-interaction-per-second**. (The wallet software of real users can be expected to select evenly/randomly from available threads to maximize the likelihood of a successful transaction.) Multiple threads can be tracked by the parent covenant (e.g. using a merkle tree), and thread check-ins can be performed incrementally, so covenants can be designed to support a practically unlimited number of threads.

Finally, exceptionally active covenant applications – or applications with the potential to incentivize spend-races – should consider using **managed threads**: threads which also require the authorization of a particular key, set of keys, or non-fungible token for each submitted state change. Managed threads allow transaction submission to be ordered without contention by the entity/entities managing each thread; they can be issued either to trusted parties or via a trustless strategy, e.g. requiring a sufficiently large deposit to disincentivize frivolous thread creation.

## Rationale

This section documents design decisions made in this specification.

### Incompatibility of Token Fungibility and Token Commitments

Advanced BCH contract use cases require strategies for transferring authenticated commitments – messages attesting to ownership, authorization, credit, debt, or other contract state – from one contract to another (a motivation behind [PMv3](https://github.com/bitjson/pmv3)). These use cases often conflicted with previous, fungibility-focused token proposals ([`OP_CHECKCOLORVERIFY`](https://bitcointalk.org/index.php?topic=253385.0), [`OP_GROUP`](https://www.bitcoinunlimited.net/grouptokenization/groupbchspec), [Unforgeable Groups](https://gitlab.com/0353F40E/group-tokenization/-/blob/master/CHIP-2021-02_Unforgeable_Groups_for_Bitcoin_Cash.md), [Confidential Assets](https://blockstream.com/bitcoin17-final41.pdf)).

One key insight which precipitated this proposal's bifurcated fungible/non-fungible approach is: **token fungibility and token commitments are conceptually incompatible**.

Fungible tokens are (by definition) indistinguishable from one another. Fungible token systems must allow amounts of tokens to be freely divided and re-merged without tracking the precise flow of individual token units. Conversely, nonfungible tokens (as defined by this proposal) are most useful to contracts because they offer a strategy for issuing tamper-proof messages which can be read and acted upon by other contracts.

Any token standard which attempts to combine these primitives must contend with their conceptually incompatibility – "fungible" tokens with commitments are not strictly fungible (e.g. some covenants could reject certain commitments, so wallet software must "assay" quantities of such tokens) and must have either implicit or user-defined policies for splitting and merging commitments (increasing protocol complexity and impeding standardization).

By clearly separating the fungible and non-fungible use cases, this specification is able to reduce each to a more fundamental, VM-compatible primitive. Rather than exhaustively specifying minting, transfer, or destruction "policies" at the protocol level – or creating another subsystem in which such policies are user-defined – all such policies can be specified using the existing Bitcoin Cash VM bytecode.

### Shared Codepoint for All Tokens

Though fungible and non-fungible tokens are entirely independent primitives, this specification defines the same `PREFIX_TOKEN` codepoint for encoding both token types.

While specifying separate "`TOKEN_FUNGIBLE`" and "`TOKEN_NONFUNGIBLE`" codepoints could save one byte for outputs which encode only a single token type, such separation would introduce significant overhead for covenants which operate on tokens of both types (within a single category): holding one token type would prevent covenants from holding the other type in the same output (assuming [outputs must have only one prefix](#one-prefix-codepoint-per-output)). While [depository covenants](#depository-child-covenants) can always be used to hold other categories of tokens, this option would still force many developers to use multi-output, "sidecar" covenant designs, even for relatively simple applications.

A third "`TOKEN_DUAL`" could add support for these covenant cases, but specifying multiple token encodings would add significant implementation cost, particularly for wallet software.

Finally, many other strategies for reducing transaction sizes can be accomplished using additional codepoints (e.g. opcodes for commonly-adjacent operations like `OP_EQUALVERIFY`), often promising far greater per-codepoint savings.

### Behavior of Minting and Mutable Tokens

This specification includes support for adding two different "capabilities" to non-fungible tokens: `minting` and `mutable`.

Minting tokens allow the holder to create new non-fungible tokens which share the same category ID as the minting token. By implementing this minting capability for a category (rather than requiring all category tokens to be minted in a single transaction), this specification enables use cases which require both a stable category ID and long-running issuance. This is critical for both primarily-off-chain applications of non-fungible tokens (where issuers commonly require both stable identifiers and an issuer-provided commitment) and for most covenant use cases (where covenants must be able to create new commitments in the course of operation).

By implementing category-minting control as a token, minting policies can be defined using complex contracts (e.g. multisig vaults with time-based fallbacks) or even covenants (e.g. to enforce uniqueness of commitments within the category). Notably, this specification even allows minting tokens to create additional minting tokens; this is valuable for [highly-specialized covenants](#multithreaded-covenants) which require the ability to delegate token-creation authority to other covenants.

Mutable tokens allow the holder to create **only one new token** (i.e. "modify" the mutable token's commitment) which may again have the mutable capability.

This is a particularly critical use case for covenants, as it enables covenants to modify the commitment in a [tracking token](#covenant-tracking-non-fungible-tokens) without exhaustively validating that the interaction did not unexpectedly mint new tokens (allowing the user to impersonate the covenant). While exhaustive validation could be made efficient with new VM opcodes, such validation may commonly conflict across covenants, preventing them from being used in the same transaction. As such, this proposal considers the `mutable` capability to be essential for [cross-covenant interfaces](#cross-contract-interfaces).

Note, because minting and mutable tokens are not immutable, they can be implicitly destroyed, i.e. non-fungible tokens with either capability are not required to be transferred from a transaction's UTXOs to its outputs for the transaction to be considered valid ([unlike most tokens](#disallowing-implicit-destruction-of-immutable-tokens)). This is an important optimization for covenant use cases – tokens controlled by a covenant can be used to hold or share internal state within a coordinating set of covenants. By not requiring the token to be duplicated to a new `OP_RETURN` output each time it is mutated, transaction sizes are significantly reduced.

### Disallowing Implicit Destruction of Immutable Tokens

This specification prevents ([most](#allowing-implicit-destruction-of-minting-tokens)) spent tokens from being implicitly destroyed by failing to re-include them in a transaction's outputs; for a transaction to be valid, each token present in the spent UTXOs must also appear in some output. This validation improves the safety of all wallet software – particularly of software released prior to the adoption of this specification (simplistic wallet software often does not parse or validate its own UTXOs, and may not be fully-aware of tokens).

Wallet software is most likely to be tested for correctness in the common case of sending and receiving BCH, but it is decreasingly likely to be free of bugs in less common functionality – like receiving and sending tokens. This specification takes the conservative approach: require tokens to be explicitly destroyed (by e.g. sending them to an `OP_RETURN` output), protecting end-users from loss due to wallet software bugs.

### Avoiding Proof-of-Work for Token Data Compression

Previous token proposals require token creators to retry hashing preimages until the resulting token category ID matches required patterns. This strategy enables additional bits of information to be packed into the category ID.

In practice, such proof-of-work strategies unnecessarily complicate covenant-managed token creation. To create [ecosystems of contracts which are mutually-aware of each other contract](#covenant-tracking-non-fungible-tokens), it is valuable to be able to predict the category ID of a token which has not yet been created; hashing strategies often preclude such planning.

Finally, at a software level, rapid retrying of hash preimages is expensive to implement and audit. Wallet software must recognize which fields may be modified, and some logic must select the parameters of each attempt. This additional flexibility presents a large surface area for exfiltration of key material (i.e. hiding parts of the private key in various modifiable structures). Signing standards for mitigating this risk may be expensive to specify and implement.

### Use of Transaction IDs as Token Category IDs

In defining token category IDs, this proposal makes a tradeoff: new token categories can only be created using outpoints with an index of `0` (UTXOs which were the 0th output of a transaction), and in exchange, **existing transaction indexes can be used to locate token genesis transactions** (token category creation transactions).

This tradeoff significantly simplifies implementations in node and indexing software, and it **allows wallet software to easily query and verify both genesis transaction information and later token history**. This can be useful for verifying token supply, inspecting data included in any genesis transaction `OP_RETURN` outputs, or performing other protocols (e.g. inspecting [Bitauth](https://github.com/bitauth/bitauth-cli) metadata).

Additionally, the one-category-per-transaction requirement is trivial to meet: token-creating wallet software can provide new category IDs using single-output (zero-confirmation) intermediate transactions; contracts which oversee token creation can require users to provide a sufficient number of category IDs (via such intermediate transactions) and need only verify that they have been provided (as the VM would reject invalid token category IDs prior to contract evaluation).

Finally, If real-world usage demonstrates demand, a future upgrade could enable token category creation with non-zero outpoint indexes by, e.g. allowing outputs with new token categories to use a hash of the full outpoint (both hash and index) in the corresponding input. (Such hashes must be 32 bytes to prevent birthday attacks.)

### One Prefix Codepoint Per Output

Use cases can be demonstrated for allowing the inclusion of multiple token prefixes within the same output (e.g. multiple non-fungible tokens, or multiple categories of fungible tokens). This specification allows only one prefix codepoint per output for several reasons:

- **More efficient indexing** – by allowing only one prefix, indexing software need not parse token prefixes to create indexes, and such non-token-aware software need only allow for a fixed prefix size (up to `84` bytes<sup>1</sup>).
- **Inspecting multiple prefixes** – a model which allows for multiple prefixes must also offer options for inspecting multiple groups of token information, increasing the required complexity of token inspection operations. This would require either additional VM codepoints or additional input items per opcode (increasing transaction sizes).

Importantly, it should be noted that alternative strategies exist for effectively locking multiple (groups of) tokens to a single output. For example, a decentralized order book for trading non-fungible tokens could hold a portfolio of tokens in separate [depository covenants](#depository-child-covenants), contracts which require the parent covenant to participate in any transactions. (The sub-covenant can likewise verify the authenticity of the parent using a ["tracking" non-fungible token](#covenant-tracking-non-fungible-tokens) which moves along with the parent covenant.) This parent-child strategy offers contracts practically unlimited flexibility in holding portfolios of both fungible and non-fungible tokens.

<details>

<summary>Notes</summary>

1. The maximum-length token prefix contains greater than `4294967295` fungible tokens and a non-fungible token (with both a capability and a 40-byte commitment): `PREFIX_TOKEN` – `1` byte; `category` – `32` bytes; `capability` – `1` byte; `commitment` length – `1` byte; `commitment` – `40` bytes; `amount` – `9` bytes; total: `84` bytes.

</details>

### Non-Fungible Token Commitment Length

Limiting non-fungible token `commitment` length is valuable because it restrains unnecessary growth of the UTXO set and limits requirements for general-purpose indexing software. This specification limits the `commitment` field of non-fungible tokens to `40` bytes.

By committing to a hash, contracts can commit to an unlimited collection of data (e.g. a merkle tree). For resistance to [birthday attacks](https://bitcointalk.org/index.php?topic=323443.0), covenants should typically avoid hashes shorter than `32` bytes. This proposal expands this minimum requirement by `8` bytes, a (maximum-size) VM number. This is particularly valuable for covenants which are optimized to use multiple types of commitment structures (e.g. re-organizing an unbalanced merkle tree of contract state for efficiency when the covenant enters "voting" mode), and need to concisely indicate their current internal "mode" to other contracts. Other valuable constructions also fit within this range: two 20-byte hashes, a 33-byte compressed public key (e.g. Schnorr public key aggregation), a hash locator for content-addressable storage and 8-byte VM number, and a 320-bit hash (if deployed by a future upgrade).

### Including Capabilities in Token Category Inspection Operations

The token category inspection operations (`OP_*TOKENCATEGORY`) in this proposal push the **concatenation of both category and capability** (if present). While token capabilities could instead be inspected with individual `OP_*TOKENCAPABILITY` operations, the behavior specified in this proposal is valuable for contract efficiency and security.

First, the combined `OP_*TOKENCATEGORY` behavior reduces contract size in the most common case: every covenant which handles tokens must regularly compare both the category and capability of tokens. With the separated `OP_*TOKENCATEGORY` behavior, this common case would require at least 3 additional bytes for each occurrence – `<index> OP_*TOKENCAPABILITY OP_CAT`, and commonly, 6 or more bytes: `<index> OP_UTXOTOKENCAPABILITY OP_CAT` and `<index> OP_OUTPUTTOKENCAPABILITY OP_CAT` (`<index>` may require multiple bytes).

There are generally two other cases to consider:

- **covenants which hold mutable tokens** (somewhat common) – these covenants are also optimized by the combined `OP_*TOKENCATEGORY` behavior. Because their mutable token can only create a single new mutable token, they need only verify that the user's transaction doesn't steal that mutable token: `OP_INPUTINDEX OP_UTXOTOKENCATEGORY <index> OP_OUTPUTTOKENCATEGORY OP_EQUALVERIFY` (saving at least 4 bytes when compared to the separated approach: `OP_INPUTINDEX OP_UTXOTOKENCATEGORY OP_INPUTINDEX OP_UTXOTOKENCAPABILITY OP_CAT <index> OP_OUTPUTTOKENCATEGORY <index> OP_OUTPUTTOKENCAPABILITY OP_CAT OP_EQUALVERIFY`).
- **covenants which hold minting tokens** (rare) – because minting tokens allow for new tokens to be minted, these covenants must exhaustively verify all outputs to ensure the user has not unexpectedly minted new tokens. (For this reason, minting tokens are likely to be held in isolated, minting-token child covenants, allowing the parent covenant to use the safer `mutable` capability.) For most outputs (verifying the output contains no tokens), both behaviors require the same bytes – `<index> OP_OUTPUTTOKENCATEGORY OP_0 OP_EQUALVERIFY`. For expected token outputs, the combined behavior requires a number of bytes similar to the separated behavior, e.g.: `<index> OP_OUTPUTTOKENCATEGORY <32> OP_SPLIT <0> OP_EQUALVERIFY <depth> OP_PICK OP_EQUALVERIFY` (combined, where `depth` holds the 32-byte category set via `OP_INPUTINDEX OP_UTXOTOKENCATEGORY <32> OP_SPLIT OP_DROP`) vs. `OP_INPUTINDEX OP_UTXOTOKENCATEGORY <index> OP_OUTPUTTOKENCATEGORY OP_EQUALVERIFY OP_INPUTINDEX OP_UTXOTOKENCAPABILITY <index> OP_OUTPUTTOKENCAPABILITY` (separated).

Beyond efficiency, this combined behavior is also **critical for the general security of the covenant ecosystem**: it makes the most secure validation (verifying both category and capability) the "default" and cheaper in terms of bytes than more lenient validation (allowing for other token capabilities, e.g. during minting).

For example, assume this proposal specified the separated behavior: if an upgradable (e.g. by shareholder vote) covenant with a tracking token is created without any other token behavior, dependent contracts may be written to check that the user has also somehow interacted with the upgradable covenant (i.e. `<index> OP_UTXOTOKENCATEGORY <expected> OP_EQUALVERIFY`). If the upgradable covenant later begins to issue tokens for any reason, a vulnerability in the dependant contract is exposed: users issued a token by the upgradable covenant can now mislead the dependent contract into believing it is being spent in a transaction with the upgradeable contract (by spending their issued token with the dependent contract). Because the dependent contract did not include a defensive `<index> OP_UTXOTOKENCAPABILITY <0xfe> OP_EQUALVERIFY` (either by omission or to reduce contract size), it became vulnerable after a "public interface change". If `OP_UTXOTOKENCATEGORY` instead uses the combined behavior (as specified by this proposal) this class of vulnerabilities is eliminated.

Finally, this proposal's combined behavior preserves two additional, unused codepoints in the Bitcoin Cash VM instruction set.

### Limitation of Fungible Token Supply

Token validation has the effect of limiting fungible token supply to the maximum VM number (`9223372036854775807`). This is important for both contract usage and the token application ecosystem.

Consider decentralized exchange covenants which support submission of new liquidity pools: if it is possible to create a fungible token output with an amount greater than the maximum VM number, there is a class of vulnerabilities by which 1) the decentralized exchange fails to validate total amounts of newly submitted assets, 2) the error-producing value becomes embedded in the the covenant, and 3) the amount invalidates important spending paths, i.e. a denial of service (possibly permanently locking all funds). Because practically all contracts which handle arbitrary fungible tokens would need to employ this type of validation, limiting amounts to `9223372036854775807` reduces the size and complexity of a wide range of contracts.

Across the wider token application ecosystem (node software, indexers, and wallet software), the limit also makes maximum token supply simpler to calculate and display in user interfaces. Arbitrary-precision arithmetic is not necessary, and total supply has a maximum digit length of `19`.

Finally, while a cap of `9223372036854775807` is likely sufficient for most cases (e.g. supporting $92 quadrillion at a precision of 1 cent), it is trivial to design contracts which consider two different token categories to be interchangeable. So while each token category is limited, a single genesis transaction can create a practically unlimited number of token categories, each with the maximum supply. Contracts can be designed to consider any of a set of token categories to be completely fungible, allowing practically unlimited "virtual" token amounts (but without requiring the wider token ecosystem to perform arbitrary-precision arithmetic or display the larger amounts in generalized user interfaces).

### Specification of Token Supply Definitions

The supply of covenant-issued tokens will often be inherently analyzable using the [supply-calculation algorithms](#fungible-token-supply-definitions) included in this specification. For example, all covenants which use a [tracking token](#covenant-tracking-non-fungible-tokens) and retain a supply of unissued tokens will have an easily-calculable [circulating supply](#circulating-supply) (without requiring wallets/indexers to understand the implementation details of the contract).

By explicitly including these supply-calculation algorithms in this specification, token issuers are more likely to be aware of this reality, choosing to issue tokens via a strategy which conforms to common user expectations (improving data availability and compatibility across the ecosystem).

## Implementations

Please see the following reference implementations for additional examples and test vectors:

[TODO: after initial public feedback]

## Stakeholders & Statements

[TODO]

## Feedback & Reviews

- [CashTokens CHIP Issues](https://github.com/bitjson/cashtokens/issues)
- [`CHIP 2021-02 CashTokens` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2022-02-cashtokens-token-primitives-for-bitcoin-cash/725)

## Changelog

This section summarizes the evolution of this document.

- **v2.0.1 – 2022-2-25** (current)
  - Expand rationale
  - Note impossibility of valid token outputs in coinbase transactions
- **v2.0.0 – 2022-2-22** ([`879c55ed`](https://github.com/bitjson/cashtokens/blob/879c55edd7e9cd6a2c2d50990d89e5cc7cb07394/readme.md))
  - Initial publication (versioning begins at v2 to differentiate from [CashTokens v1](https://blog.bitjson.com/cashtokens-contract-validated-tokens-for-bitcoin-cash/))

## Copyright

This document is placed in the public domain.
