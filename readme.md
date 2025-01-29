# CHIP-2025-01 TXv5: Transaction Version 5

        Title: Transaction Version 5
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2025-01-28
        Latest Revision Date: 2025-01-28
        Version: 1.0.0

<summary><strong>Table of Contents</strong></summary>

- [Summary](#summary)
- [Benefits](#benefits)
- [Deployment](#deployment)
- [Technical Specification](#technical-specification)
- [Rationale](#rationale)
<!-- - [Implementations](#implementations) -->
- [Feedback \& Reviews](#feedback--reviews)
- [Changelog](#changelog)
- [Copyright](#copyright)

</details>

## Summary

This proposal introduces a version 5 transaction format, enables fractional satoshis and fractional CashTokens, simplifies contracts, and optimizes transaction sizes.

## Benefits

- **Zero-Overhead Covenants** – Enable user-deployed financial and privacy covenants to minimize transaction sizes, matching or outperforming purpose-built, "layer 1" networks. See [Example: Zero-Knowledge Proof Covenants](#zero-knowledge-proof-covenants).
- **Detached ("Cross-Input") Signatures** – Reduce transaction sizes by enabling multiple inputs to reference the same signature(s). See [Example: Single-Signature Sweeps](#single-signature-sweeps).
- **Comprehensive Malleability Protection** – Optionally covers all unlocking bytecode, simplifying security audits and allowing contracts to eliminate piecemeal protections. Non-push unlocking bytecode also enables significant savings for transactions spending multiple similar inputs.
- **Efficient Covenant UTXO Recycling** – Enables the economical inclusion of recycling code paths in covenants, allowing covenant systems to recover dust from obsolete UTXOs. See [Example: Recycling Covenant Dust](#recycling-covenant-dust).
- **Fractional Satoshis** – Prepare Bitcoin Cash for significantly increased scale and adoption (while saving up to 6 bytes per output vs. the existing encoding).
- **Fractional CashTokens** – Simplify the operational requirements of large, highly-liquid CashTokens like stablecoins and real-world assets by optionally allowing up to 19 decimal places of additional precision.

## Examples

The following examples highlight efficiency improvements made possible by v5 transactions.

### Zero-Knowledge Proof Covenants

**This proposal enables zero-overhead covenants: user-deployed protocols with the same bandwidth efficiency as "layer 1" networks.**

For example, a zero-knowledge proof (ZKP) contract system could be designed to perform validation of proofs within a [read-only input](#read-only-inputs) known to the covenant (by outpoint) or marked by a [tracking CashToken](https://cashtokens.org/docs/spec/examples#covenant-tracking-identity-tokens). Such ZKP transactions might include three inputs and two outputs:

- `Input 0` (read-only): The ZKP is provided in this unlocking bytecode. The locking bytecode verifies that the transaction is correctly structured and that the ZKP justifies the state transition between the previous application state (`Input 1`) and the next application state (`output 0`).
- `Input 1`: A covenant holding the previous application state and any BCH, spendable only if `Input 0` is the expected ZKP-verifying contract. (Any combination of other tokens may be controlled by sidecar UTXOs, omitted or included as necessary for deposits/withdrawals.)
- `Input 2`: A user-provided input to fund any deposit(s) and the transaction's mining fee.
- `Output 0`: The updated covenant holding the new application state and any BCH.
- `Output 1`: A user-controlled output to hold any withdrawals or remaining "change" in BCH (if `Input 2` isn't entirely consumed).

Note the bandwidth-efficiency of this transaction: it encodes 1) deposited/withdrawn asset amounts and details 2) the covenant's "after" state (the "before" state is known via `Input 1`), and 3) the proof. In particular, both the covenants "before" state and the bytecode evaluated to verify the proof – possibly many hundreds or thousands of bytes – is omitted from the transaction (and de-duplicated in the blockchain).

In short, this ZKP protocol is just as bandwidth-efficient as if it were "built-in" to Bitcoin Cash via consensus upgrade.

Following activation of this proposal, **Bitcoin Cash covenants could simultaneous support a wide variety of financial and privacy technologies using transaction sizes no larger than those of purpose-built, "layer 1" networks**.

Further, Bitcoin Cash covenants are better positioned to safely adopt new technologies: other networks require extensive development, testing, and advocacy prior to the deployment of upgrades, while equally-efficient Bitcoin Cash covenants can be deployed immediately, at negligible cost, without risk to the wider Bitcoin Cash network, and with risks precisely controlled by the end user (i.e. only the minimal assets trusted to a new system can be lost by a vulnerability in that system).

Finally, deployed technologies on other networks often trail significantly behind the "state of the art" (e.g. in proof sizes). Zero-overhead covenants allow Bitcoin Cash to consistently outperform these other networks in transaction sizes, scalability, and overall user experiences.

### Single-Signature Sweeps

**This proposal enables significant reductions in common transaction sizes by sharing signatures across inputs and re-enabling non-push unlocking bytecode.**

For example, a transaction spending many Pay-to-Public-Key-Hash (P2PKH) outputs may unlock the 0th input with unlocking bytecode: `OP_0 <a.public_key>` (35 bytes), where the `OP_0` indicates the 0th [detached signature](#detached-signatures). Later inputs may then be spent simply by evaluating the push instructions from the 0th input: `OP_0 OP_INPUTBYTECODE OP_EVAL` (3 bytes per input).

Once constructed, the entire encoded transaction can be signed only once (by `a.public_key`), with the signature placed in the 0th detached signature slot, preventing malleation of any portion of the transaction.

### Recycling Covenant Dust

This proposal resolves a long-standing incentive alignment issue in contract design: covenant UTXOs which have reached the end of their lifecycle (and in many cases, may be provably unspendable) often remain in the UTXO set for lack of a fee-efficient recycling code path.

Prior to this proposal, the aggregate cost of replicating a recycling code path across a covenant's lifecycle often exceeded the recovered value, even for very short bytecode lengths.

For example, consider this "wrapper" allowing a covenant UTXO to be recycled once its value reaches some minimum value (`600` sats): `OP_INPUTINDEX OP_UTXOVALUE <600> OP_LESSTHAN OP_NOTIF other_contract_code OP_ENDIF` (adds 8 bytes). If this covenant were re-created across a chain of 100 transactions, the transaction fees required (currently 1 byte/sat) for the cumulative overhead contributed by the wrapper (800 bytes) would exceed the expected value of the recovered dust.

Following this proposal, any covenant can be designed such that the aggregate cost of recycling code paths are small and constant: by including recycling code path(s) in [read-only inputs](#read-only-inputs), fees paid for inclusion of recycling bytecode are limited to the initial setup transaction(s). As recycling bytecode can be omitted from all later transactions, covenant dust can be economically collected in nearly all cases.

### High-Liquidity CashTokens

Prior to this proposal, high-liquidity CashTokens (particularly stablecoins) were forced to navigate a minor tradeoff in denominating their CashToken: larger units reduce the encoding size in most transactions and enable a larger overall supply, but smaller units may be necessary for sufficient precision in important markets and use cases.

For example, using indivisible CashTokens, a USD-backed stablecoin issuer might denominate its stablecoin using 6 decimal places (i.e. `1000000` fungible tokens represent `$1`) – sufficient precision to accommodate the minimum tick size in today's largest financial markets (allowing on-chain settlements without batching or precision-handling edge cases).

However, at 6 decimal places, nearly all common output amounts require 5 bytes (e.g. $1 is `0xfe40420f00`) or more, whereas at 2 decimal places, many smaller output amounts require only 1 byte (e.g. `$1` is `0x64`) or 3 bytes (e.g. `$655.35` is `0xfdffff`). Additionally, at 6 decimal places, the maximum indivisible fungible token supply can represent only $9.2 trillion USD, leaving less than one order of magnitude between the current "market cap" of all cryptocurrencies and the stablecoin's maximum-supportable supply. This leaves room for plausible concern about migration strategies and/or multi-category issuance schemes, particularly in the event of rapid inflation in the underlying fiat currency.

Following this proposal, this minor tradeoff is alleviated: in addition to the issuer's chosen number of decimal places, **up to 19 decimal places of additional precision are available for all [fractional CashTokens](#fractional-cashtokens)**.

## Deployment

Deployment of this specification is proposed for the May 2027 upgrade.

- Activation is proposed for `1794744000` MTP, (`2026-11-15T12:00:00.000Z`) on `chipnet`.
- Activation is proposed for `1810382400` MTP, (`2027-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, `testnet4`, and `scalenet`.

## Technical Specification

### Transaction

The following table compares the existing transaction format with the v5 transaction format.

| Field                    | v1/v2 Format                         | v5 Format                                   | Description                                                           |
| ------------------------ | ------------------------------------ | ------------------------------------------- | --------------------------------------------------------------------- |
| Version                  | Uint32 (4&nbsp;bytes)                | Compact UInt (1&nbsp;byte)                  | The version of the transaction format (`0x05`).                       |
| Input Count              | Compact UInt (1&#x2011;4&nbsp;bytes) | Compact UInt (1&#x2011;4&nbsp;bytes)        | The number of inputs in the transaction.                              |
| Transaction Inputs       | [Inputs](#transaction-input)         | [Inputs](#transaction-input)                | Each of the transaction’s inputs serialized in order.                 |
| Output Count             | Compact UInt (1&#x2011;4&nbsp;bytes) | Compact UInt (1&#x2011;4&nbsp;bytes)        | The number of outputs in the transaction.                             |
| Transaction Outputs      | [Outputs](#transaction-output)       | [Outputs](#transaction-output)              | Each of the transaction’s outputs serialized in order.                |
| Locktime                 | Uint32 (4&nbsp;bytes)                | Uint32 (4&nbsp;bytes)<sup>1</sup>           | A time or block height at which the transaction is considered valid.  |
| **Optional Fields**      |
| Detached Signature Count | N/A                                  | Compact UInt (1&#x2011;4&nbsp;bytes)        | The number of detached signatures in the transaction. Omitted if `0`. |
| Detached Signatures      | N/A                                  | [Detached Signatures](#detached-signatures) | Each of the transaction's detached signatures serialized in order.    |

<details>
 <summary>Notes</summary>

1. The format of the Locktime field is unmodified to avoid disincentivizing [an existing strategy for block re-organization resistance](https://bitcoincashresearch.org/t/chip-2021-01-pmv3-version-3-transaction-format/265/48?u=bitjson#how-transaction-formats-can-interact-with-mining-incentives-4).

</details>

### Detached Signatures

Detached signatures are signatures which use the [`SIGHASH_DETACHED` signing serialization algorithm](#sighash_detached-algorithm) (based on the v5 transaction encoding). The `Detached Signatures` field is an ordered list of these signatures. Each detached signature is prefixed with a length:

| Field                     | v1/v2 Format | v5 Format                  | Description                                                                                                                                                                                     |
| ------------------------- | ------------ | -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Detached Signature Length | N/A          | Compact UInt (1&nbsp;byte) | The byte-length of the following detached signature field. Currently limited to 73 bytes.                                                                                                       |
| Detached Signature        | N/A          | Signature (64+ bytes)      | The detached signature (either [BIP66](https://github.com/bitcoin/bips/blob/65529b12bb01b9f29717e1735ce4d472ef9d9fe7/bip-0066.mediawiki) ECDSA or Schnorr) including the sighash byte (`0x40`). |

By committing to the entire encoded transaction – including the contents of all inputs – **detached signatures comprehensively eliminate third-party malleability**. Applications may also use detached signatures to reduce transaction sizes by deduplicating signatures (as multiple inputs may reference the same detached signature).

Detached signatures are referenced by their index from unlocking bytecode using [signature references](#signature-references).

By consensus:

1. Detached signatures may appear in any order, but no signature may be duplicated in the `Detached Signatures` field (as this would introduce a malleability vector).
2. Each detached signature must be referenced at least once during transaction validation.
3. Detached signatures are currently limited to 73 bytes (to accommodate existing ECDSA and Schnorr signatures), but future upgrades may allow for longer signature types.
4. Use of detached signatures in v5 transactions is optional, but the encoding of transactions which include no detached signatures must end with the `Locktime` field, i.e. the `Detached Signature Count` must never be `0x00` (transactions without detached signatures must omit the field).
5. All detached signatures must set `SIGHASH_FORKID` (`0x40`), and no other flags are currently permitted.<sup>1</sup>

Note that detached signatures are "detached" from the inputs which use them, allowing them to be excluded from their own signing serialization (signatures cannot sign themselves). However, detached signatures are **included** in the transaction's hash/`TXID` for `Outpoint Transaction Hash`, block merkle trees, and for other P2P messages.

<details>
 <summary>Notes</summary>

1. Note, the `SIGHASH_DETACHED` algorithm is not represented by a sighash byte, as it is currently the only valid algorithm for detached signatures. Future upgrades may specify other algorithms and valid sighash byte values for detached signatures. (E.g. non-interactive cross-input aggregation schemes, opt-in replay protection, etc.)

</details>

#### `SIGHASH_DETACHED` Algorithm

A new [signing serialization algorithm](https://github.com/bitcoincashorg/bitcoincash.org/blob/3e2e6da8c38dab7ba12149d327bc4b259aaad684/spec/replay-protected-sighash.md#specification), `SIGHASH_DETACHED`, is used for all detached signatures. The algorithm is equivalent to the [version 5 transaction encoding](#transaction) prefixed with the [Fork ID](https://reference.cash/protocol/forks/replay-protected-sighash) and truncated immediately after `Locktime`. See [Rationale: Retention of Fork ID](#retention-of-fork-id).

| Field                 | Format                                 | Description                                         |
| --------------------- | -------------------------------------- | --------------------------------------------------- |
| Fork ID               | Fork ID (3&nbsp;bytes)                 | On BCH, `0x000000`.                                 |
| Truncated Transaction | [v5 Encoded Transaction](#transaction) | The encoded transaction truncated after `Locktime`. |

To reference a detached signature from bytecode, the index of the referenced signature is encoded as a VM Number. This `signature reference` is pushed in place of a standard signature for the `OP_CHECKSIG`, `OP_CHECKSIGVERIFY`, `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY` operations.

In each signature-checking operation, when the signature popped from the stack is found to have a length less than or equal to `2` bytes, the reference is parsed as an index (VM Number), and the detached signature at that index is used for the remainder of the signature checking operation. (This allows for indexes up to `32767`, far beyond existing limits to transaction size.)

#### Signature References

Detached signatures are referenced by pushing a valid detached signature index (as a `Script Number`) in place of a signature. Because the VM includes single-byte operations from `OP_0` to `OP_16`, up to 16 detached signatures can be referenced using a single byte.

<details>
<summary>Signature Reference Test Vectors</summary>

| Encoded      | Disassembled              | Script Number | Detached Signature Index |
| ------------ | ------------------------- | ------------- | ------------------------ |
| `0x00`       | `OP_0`                    | `0x`          | `0`                      |
| `0x51`       | `OP_1`                    | `0x01`        | `1`                      |
| `0x52`       | `OP_2`                    | `0x02`        | `2`                      |
| `0x60`       | `OP_16`                   | `0x10`        | `16`                     |
| `0x0111`     | `OP_PUSHBYTES_1 0x11`     | `0x11`        | `17`                     |
| `0x017f`     | `OP_PUSHBYTES_1 0x7f`     | `0x7f`        | `127`                    |
| `0x028000`   | `OP_PUSHBYTES_2 0x8000`   | `0x8000`      | `128`                    |
| `0x03ff7f`   | `OP_PUSHBYTES_2 0xff7f`   | `0xff7f`      | `32767`                  |
| `0x04008000` | `OP_PUSHBYTES_3 0x008000` | `0x008000`    | Invalid                  |

</details>

### Input

The following table compares the existing input format with the v5 input format.

| Field                     | v1/v2 Format                         | v5 Format                                       | Description                                                                                                                                                                                                        |
| ------------------------- | ------------------------------------ | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Outpoint Transaction Hash | hash (32&nbsp;bytes)                 | hash (32&nbsp;bytes)                            | The hash of the transaction containing the output being spent.                                                                                                                                                     |
| Outpoint Index            | Uint32 (4&nbsp;bytes)                | Compact UInt (1&#x2011;3&nbsp;bytes)            | The zero-based index of the output being spent from the previous transaction.                                                                                                                                      |
| Unlocking Bytecode Length | Compact UInt (1&#x2011;5&nbsp;bytes) | Compact UInt (1&#x2011;5&nbsp;bytes)            | The size of the unlocking script in bytes.                                                                                                                                                                         |
| Unlocking Bytecode        | bytecode                             | bytecode                                        | The unlocking bytecode.                                                                                                                                                                                            |
| Input Bitfield            | Sequence Number High Bits (2 bytes)  | [Input Bitfield](#input-type) (1 byte)          | As of [BIP68](https://github.com/bitcoin/bips/blob/65529b12bb01b9f29717e1735ce4d472ef9d9fe7/bip-0068.mediawiki), sequence number is a complex bitfield primarily interpreted as a relative locktime for the input. |
| Age Lock                  | Sequence Number Low Bits (2 bytes)   | (optional) Compact UInt (1&#x2011;3&nbsp;bytes) | Included in v5 format only if enabled by the [`Input Bitfield`](#input-type).                                                                                                                                      |

#### Non-Push Unlocking Bytecode

Unlocking bytecode encoded in the v5 transaction format is not limited to push-only operations (A.K.A. `SCRIPT_VERIFY_SIGPUSHONLY`), as [detached signatures](#detached-signatures) offer comprehensive malleability protection.

#### Maximum Bytecode Length

The existing 10,000-byte limit on maximum bytecode length (A.K.A. `MAX_SCRIPT_SIZE`) is raised to 100,000 bytes, equal to the limit on maximum standard transaction byte length (A.K.A. `MAX_STANDARD_TX_SIZE`). See [Rationale: Unification of Byte Length Limits](#unification-of-byte-length-limits).

#### Maximum Stack Element Length

The existing 10,000-byte stack element length limit (A.K.A. `MAX_SCRIPT_ELEMENT_SIZE`) is raised to 100,000 bytes, equal to the limit on maximum standard transaction byte length (A.K.A. `MAX_STANDARD_TX_SIZE`). See [Rationale: Unification of Byte Length Limits](#unification-of-byte-length-limits).

#### Input Bitfield

Rather than the existing 4-byte Sequence Number encoding, v5 transaction inputs encode the same information using a backwards-compatible, 1-byte `Input Bitfield` with an optional `Age Lock` (Compact UInt).

**The `Input Bitfield` format is exclusively an encoding format** – all rules and behaviors relating to sequence number continue to be enforced based on the encoded sequence number.

| Bit(s)  | Name                | Description                                                                                                                                                                                                                                                     |
| ------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0`     | Enable Locktime     | If unset, Locktime is disabled for this input, sequence number is exactly `0xffffffff` (`4294967295`), and no other bits may be set (invalid by consensus). See [BIP65 for details](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki).            |
| `1`     | Enable Age Lock     | If set, the next byte begins the `Age Lock` field (Compact UInt).                                                                                                                                                                                               |
| `2`     | Time-Based Age Lock | If set, the `Age Lock` field is denoted in units of `512` seconds rather than blocks, and bit `1` must also be set (by consensus). (See [BIP68 for details](https://github.com/bitcoin/bips/blob/65529b12bb01b9f29717e1735ce4d472ef9d9fe7/bip-0068.mediawiki).) |
| `3`     | Read-Only Input     | If set, this input is read-only. (See [Read-Only Inputs](#read-only-inputs).)                                                                                                                                                                                   |
| `4`-`7` | Unassigned          | If set, these bits render the transaction invalid (by consensus). Future upgrades may assign behaviors to these bits.                                                                                                                                           |

#### Computing Sequence Number

To compute a v5 transaction input's sequence number, input bitfield elements are mapped to the v1/v2 sequence number encoding:

- `Enable Locktime` – if unset, sequence number is `0xffffffff` (`4294967295`). See [BIP65](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki).
- `Enable Age Lock` – if set, sequence number bit 31 is set (A.K.A. "Disable Flag"). See [BIP68](https://github.com/bitcoin/bips/blob/65529b12bb01b9f29717e1735ce4d472ef9d9fe7/bip-0068.mediawiki).
- `Time-Based Age Lock` – if set, sequence number bit 22 is set (A.K.A. "Type Flag"). See [BIP68](https://github.com/bitcoin/bips/blob/65529b12bb01b9f29717e1735ce4d472ef9d9fe7/bip-0068.mediawiki).
- `Read-Only Input` – if set, sequence number bit 23 is set. See [Read-Only Inputs](#read-only-inputs).
- `Age Lock` – Mapped to the lowest 16 bits of sequence number. See [BIP68](https://github.com/bitcoin/bips/blob/65529b12bb01b9f29717e1735ce4d472ef9d9fe7/bip-0068.mediawiki).

Note that the behavior of `OP_INPUTSEQUENCENUMBER` is indistinguishable between v1/v2 and v5 transaction inputs – regardless of encoding, the computed sequence number is pushed to the stack.

#### Read-Only Inputs

Following activation of this proposal, all inputs in which sequence number bit `23` is set (in [Input Bitfield](#input-bitfield) encoding, bit `3`) are interpreted as read-only inputs.<sup>1</sup>

Read-only inputs enable contract systems to deduplicate bytecode across transactions by eliminating the need to re-create equivalent outputs for repeated evaluations. This allows commonly used UTXOs to be simultaneously referenced/evaluated – but not spent – by multiple transactions.

Read-only inputs are validated as normal with one modification: assets held within read-only inputs are excluded from the sum(s) of assets which may be reassigned by the transaction (including for transaction fees).

Following validation, the UTXO referenced by the read-only input is not marked as spent by the node implementation. Note that some UTXOs may be referenced/evaluated any number of times within the same block in which they are ultimately spent.

Transactions containing only read-only inputs are invalid (by consensus).<sup>2</sup>

<details>
 <summary>Notes</summary>

1. Applications relying on read-only input behavior immediately following the upgrade may either use v5 transactions or (for v1/v2 transactions) set a sufficient locktime to guarantee enforcement of read-only input behavior within the viable block re-organization window.
2. Note that these transactions would otherwise be nonstandard as they must necessarily omit any transaction fee (given the absence of funds available to spend).

</details>

#### Nonstandard Sequence Numbers

In v1 and v2 transactions, sequence numbers which cannot be encoded in v5 transactions (i.e. using unassigned sequence number bits) render the transaction nonstandard. Users are advised that nonstandard/unassigned sequence number bits may be assigned or consensus-invalidated by future upgrades.

### Output

The following table compares the existing output format with the v5 output format.

| Field                        | v1/v2 Format                         | v5 Format                                                                          | Description                              |
| ---------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------- | ---------------------------------------- |
| Value (Satoshis)             | Uint64 (8&nbsp;bytes)                | [Compact UInt + Compact UInt](#fractional-asset-encoding) (2&#x2011;18&nbsp;bytes) | The value transferred.                   |
| Locking Bytecode Length      | Compact UInt (1&#x2011;3&nbsp;bytes) | Compact UInt (1&#x2011;3&nbsp;bytes)                                               | The byte-length of the locking bytecode. |
| (optional) Token Prefix      | Token Prefix                         | Token Prefix                                                                       | The (optional) token prefix.             |
| Locking Bytecode<sup>1</sup> | bytecode                             | bytecode                                                                           | The locking bytecode.                    |

<details>
 <summary>Notes</summary>

1. Note the increased limits on [Maximum Bytecode Length](#maximum-bytecode-length) and [Maximum Stack Element Length](#maximum-stack-element-length) apply equally to locking and unlocking bytecode.

</details>

#### Fractional Satoshis

All v5 transaction outputs encode fractional satoshi values using [Fractional Asset Encoding](#fractional-asset-encoding).

#### Fractional Asset Encoding

Fractional asset encoding optimizes the compression of integer values and evenly-divided fractions (i.e. divided between inputs and/or miner fee) using a concatenation of two `Compact UInt`s. See [Rationale: Optimization of Evenly-Divided Fractions](#optimization-of-evenly-divided-fractions).

Fractional asset encoding also maximizes the encodable precision – **to approximately 19 decimal places** – without introducing a new numeric encoding or requiring >64-bit math operations for input/output balancing (fractions can be summed using only overflow-detecting 64-bit operations).

| Field            | Format                               | Description                                                                                             |
| ---------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| Whole Units      | Compact UInt (1&#x2011;9&nbsp;bytes) | A simple integer representing whole units (of satoshis or cashtokens).                                  |
| Fractional Value | Compact UInt (1&#x2011;9&nbsp;bytes) | The numerator and denominator of the fractional value. If `0` (`0x00`), no fractional value is encoded. |

For `Fractional Value`s, the denominator is derived from the length of the `Compact UInt`:

| `Compact UInt` Length            | Decimal Range                       | Denominator            |
| -------------------------------- | ----------------------------------- | ---------------------- |
| **1&nbsp;byte** (no prefix)      | `0`–`252`                           | `256`                  |
| **2&nbsp;bytes** (`0xfd` prefix) | `253`–`65535`                       | `65536`                |
| **4&nbsp;bytes** (`0xfe` prefix) | `65536`–`4294967295`                | `16777216`             |
| **8&nbsp;bytes** (`0xff` prefix) | `4294967296`–`18446744073709551615` | `18446744073709551616` |

Fractional values must be minimally encoded (by consensus), e.g. `0.5` must be encoded as `0x80` (`128/256`) rather than `0xfd0080` (`32768/65536`).

<details>
<summary>Fractional Satoshi Test Vectors</summary>

##### Valid Fractional Values

| Hex Encoding           | Fraction                                  | Decimal      | Explanation                         |
| ---------------------- | ----------------------------------------- | ------------ | ----------------------------------- |
| `0x00`                 | 0                                         | 0            | no fractional amount                |
| `0xff0100000000000000` | 1/18446744073709551616                    | ~0.000...001 | ~`.000...1` after 19 decimal places |
| `0xfe01000000`         | 1/16777216                                | ~0.000...001 | ~`.000...1` after 7 decimal places  |
| `0xfd0100`             | 1/65536                                   | ~0.000001    | ~`.000...1` after 5 decimal places  |
| `0x1a`                 | 26/256                                    | 0.1015625    | ~`.10` to 2 decimal places          |
| `0xfd9a19`             | 6554/65536                                | 0.1000061035 | ~`.10...0` to 4 decimal places      |
| `0xfe99999919`         | 429496730/4294967296                      | ~0.1000...   | ~`.10...0` to 9 decimal places      |
| `0xff9a99999999999919` | 1844674407370955162/18446744073709551616  | ~0.1000...   | ~`.10...0` to 19 decimal places     |
| `0x01`                 | 1/256                                     | 0.00390625   | Smallest 1-byte value               |
| `0x02`                 | 2/256                                     | 0.0078125    | 1/128                               |
| `0x04`                 | 4/256                                     | 0.015625     | 1/64                                |
| `0x08`                 | 8/256                                     | 0.03125      | 1/32                                |
| `0x10`                 | 16/256                                    | 0.0625       | 1/16                                |
| `0x20`                 | 32/256                                    | 0.125        | 1/8                                 |
| `0x40`                 | 64/256                                    | 0.25         | 1/4                                 |
| `0x80`                 | 128/256                                   | 0.5          | 1/2                                 |
| `0xc0`                 | 192/256                                   | 0.75         | 3/4                                 |
| `0xe0`                 | 224/256                                   | 0.875        | 7/8                                 |
| `0xf0`                 | 240/256                                   | 0.9375       | 15/16                               |
| `0xf8`                 | 248/256                                   | 0.96875      | 31/32                               |
| `0xfc`                 | 252/256                                   | 0.984375     | 63/64                               |
| `0xfd00fe`             | 65024/65536                               | 0.9921875    | 127/128                             |
| `0xfd00ff`             | 65280/65536                               | 0.99609375   | 255/256                             |
| `0xfdffff`             | 65535/65536                               | ~0.9999      | ~`.9` to 4 decimal places           |
| `0xfeffffff00`         | 16777215/16777216                         | ~0.9999999   | ~`.9` to 7 decimal places           |
| `0xffffffffffffffffff` | 18446744073709551615/18446744073709551616 | ~0.999...    | ~`.9` to 19 decimal places          |

##### Invalid Fractional Values

| Hex Encoding           | Fraction                                                      | Decimal    | Explanation                                             |
| ---------------------- | ------------------------------------------------------------- | ---------- | ------------------------------------------------------- |
| `0xfd0080`             | 32768/65536                                                   | 0.5        | Invalid: not minimally encoded (should be `80`)         |
| `0xfe00008000`         | 8388608/16777216                                              | 0.5        | Invalid: not minimally encoded (should be `80`)         |
| `0xff0000000000000080` | 9223372036854775808/18446744073709551616                      | 0.5        | Invalid: not minimally encoded (should be `80`)         |
| `0xfe0000fe00`         | 16646144/16777216 (65024/65536; 127/128)                      | ~0.99      | Invalid: not minimally encoded (should be `fd00fe`)     |
| `0xfe00ffff00`         | 16776960/16777216 (65535/65536)                               | ~0.9999    | Invalid: not minimally encoded (should be `fdffff`)     |
| `0xff0000000000ffffff` | 18446742974197923840/18446744073709551616 (16777215/16777216) | ~0.9999999 | Invalid: not minimally encoded (should be `feffffff00`) |

</details>

#### Fractional CashTokens

The `RESERVED_BIT` in [Token Prefix encoding](https://github.com/cashtokens/cashtokens?#token-prefix) is redefined to `HAS_FRACTION`, and token prefixes with `HAS_FRACTION` enabled are considered **fractional CashTokens** as opposed to **indivisible CashTokens**.

The Token Prefix encoding is modified for fractional CashTokens by replacing the encoding of `ft_amount` with [Fractional Asset Encoding](#fractional-asset-encoding) – two `Compact UInt`s rather than one `Compact UInt`.

Any CashToken category may optionally be created ("minted") with `HAS_FRACTION` enabled. If any, all token prefixes for a particular category must enable `HAS_FRACTION`.

When spending CashTokens, the `HAS_FRACTION` setting of each token category must be preserved (by consensus), i.e. all token categories must be designated as fractional or indivisible upon creation. See [Rationale: Continued Value of Indivisible CashTokens](#continued-value-of-indivisible-cashtokens).

#### Existing Introspection Opcodes

This specification has no impact on the contract-observed behavior of `OP_UTXOVALUE` (`0xc6`), `OP_OUTPUTVALUE` (`0xcc`), `OP_UTXOTOKENAMOUNT` (`0xd0`), or `OP_OUTPUTTOKENAMOUNT` (`0xd3`): these operations continue to push an integer value with any fractional value truncated (i.e. rounded down). As such, the maximum theoretical sum of values produced by any of these opcodes for any particular asset remains equal to the maximum 8-byte VM number (`9223372036854775807`). See [Rationale: Existing vs. High-Precision Introspection Opcodes](#existing-vs-high-precision-introspection-opcodes).

For the avoidance of doubt, revised descriptions are provided below.

| Name                   | Codepoint      | Description                                                                                                                                                                                                                                                                                                                                        |
| ---------------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OP_UTXOVALUE`         | `0xc6` (`198`) | Pop the top item from the stack as an input index (VM Number). Push the truncated integer value (in satoshis) of the Unspent Transaction Output (UTXO) spent by that input to the stack as a VM Number. Note that fractional satoshi amounts are ignored by this operation.                                                                        |
| `OP_OUTPUTVALUE`       | `0xcc` (`204`) | Pop the top item from the stack as an input index (VM Number). Push the truncated integer value (in satoshis) of the output at that index to the stack as a VM Number. Note that fractional satoshi amounts are ignored by this operation.                                                                                                         |
| `OP_UTXOTOKENAMOUNT`   | `0xd0` (`208`) | Pop the top item from the stack as an input index (VM Number). Push the truncated integer amount of fungible tokens in the Unspent Transaction Output (UTXO) spent by that input to the stack as a VM Number. If the UTXO includes no fungible tokens, push a 0 (VM Number). Note that fractional cashtoken amounts are ignored by this operation. |
| `OP_OUTPUTTOKENAMOUNT` | `0xd3` (`211`) | Pop the top item from the stack as an output index (VM Number). Push the truncated integer amount of fungible tokens in the output at that index to the stack as a VM Number. If the output includes no fungible tokens, push a 0 (VM Number). Note that fractional cashtoken amounts are ignored by this operation.                               |

#### High-Precision Introspection Opcodes

Four new introspection opcodes are introduced to push high-precision satoshi values and cashtoken amounts. See [Rationale: Existing vs. High-Precision Introspection Opcodes](#existing-vs-high-precision-introspection-opcodes).

| Name                          | Codepoint      | Description                                                                                                                                                                                                                                                                                                                                                                         |
| ----------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OP_UTXOVALUEPRECISE`         | `0xd4` (`212`) | Pop the top item from the stack as an input index (VM Number). Push the high-precision value (in `2^-64`th satoshis) of the Unspent Transaction Output (UTXO) spent by that input to the stack as a VM Number. Note that the result of this operation is `18446744073709551616 * value_in_satoshis`.                                                                                |
| `OP_OUTPUTVALUEPRECISE`       | `0xd5` (`213`) | Pop the top item from the stack as an input index (VM Number). Push the high-precision value (in `2^-64`th satoshis) of the output at that index to the stack as a VM Number. Note that the result of this operation is `18446744073709551616 * value_in_satoshis`.                                                                                                                 |
| `OP_UTXOTOKENAMOUNTPRECISE`   | `0xd6` (`214`) | Pop the top item from the stack as an input index (VM Number). Push the high-precision amount (in `2^-64`ths) of fungible tokens in the Unspent Transaction Output (UTXO) spent by that input to the stack as a VM Number. If the UTXO includes no fungible tokens, push a 0 (VM Number). Note that the result of this operation is `18446744073709551616 * fungible_token_amount`. |
| `OP_OUTPUTTOKENAMOUNTPRECISE` | `0xd7` (`215`) | Pop the top item from the stack as an output index (VM Number). Push the high-precision amount (in `2^-64`ths) of fungible tokens in the output at that index to the stack as a VM Number. If the output includes no fungible tokens, push a 0 (VM Number). Note that the result of this operation is `18446744073709551616 * fungible_token_amount`.                               |

## Rationale

### Unification of Byte Length Limits

Following the [VM Limits CHIP](https://github.com/bitjson/bch-vm-limits), bytecode and stack item length limits are no longer relevant to worst-case transaction or block validation performance. As standard transactions can include many inputs – up to the maximum standard transaction byte length (A.K.A. `MAX_STANDARD_TX_SIZE`; 100,000 bytes) – lower per-item or per-input length limits offer no additional safety to the network while inconveniencing applications with larger contiguous data requirements.

For example, many zero-knowledge and post-quantum cryptographic systems require proofs larger than 10,000 bytes; with a lower per-input and/or per-item limit, these proofs must be broken into multiple stack items and/or inputs, requiring the development of unusual standards, significant waste in data manipulation bytecode, and unnecessary contract complexity.

In practice this unification saves ~40-75 bytes per 10,000 bytes by eliminating the need for bytecode and data to be divided among push operations and adjacent inputs.

### Retention of Fork ID

The proposed `SIGHASH_DETACHED` algorithm retains the [Fork ID](https://reference.cash/protocol/forks/replay-protected-sighash) feature – currently present across all Bitcoin Cash signing serialization algorithms – to minimize potential future incompatibilities between the algorithms. Additionally, while the Fork ID feature has gone unused by past chain splits, its continued existence [disincentivizes both network splits and censorship](https://x.com/bitjson/status/1305533593154920449), particularly given the credible threat of strategic use (e.g. [AllowReplay](https://github.com/bitjson/allow-replay-spec)).

### Optimization of Evenly-Divided Fractions

This proposal's [Fractional Asset Encoding](#fractional-asset-encoding) optimally-minimizes the encoded byte length of evenly-divided fractions. **This behavior is ideal for Bitcoin Cash's UTXO model, where splitting and merging of UTXO values naturally favors a radix of 2**.

When summing values, the resulting value will naturally be equally or more efficiently encoded, with base-2 maximizing the frequency of optimization, e.g. two UTXOs of `0xfd8000` satoshis (`128/65536` ~= `0.002`) sum to `0x01` satoshis (`1/256` ~= `0.004`). Larger radixes experience this optimization less frequently (based on their distance from `2`).

In the case of splitting values, it's most useful to analyze the treatment of mining fees, a primary use case for sub-satoshi division. In these cases again, base-2 offers the maximum precision per encoding cost: for any incremental increase in a transaction's mining fee paid, the minimally-encoded increase is one half unit (subtracted from one of the fee paying inputs, where base-2 encoding maximizes the frequency at which the resulting encoding length can avoid an increase in its own encoding precision). Additionally, note again the merging advantage afforded by base-2 mining fee values: given many base-2 mining fee values, the sum earned by the miner will more frequently be optimized by base-2 encoding in the earned output, as its greatest common denominator will more frequently be divisible by 2 than any larger radix.

As fungible CashTokens are not directly payable as mining fees, the treatment of mining fees is less relevant to fractional CashToken encoding (with the minor exception of "gas station" contracts that can fund mining fees in exchange for fractional tokens). However, base-2 encoding also remains optimal for fractional CashToken encoding: 1) merging of fractional token amounts (like satoshis) favors base-2, 2) consistent encoding across both satoshis and fractional CashTokens reduces protocol complexity, and 3) optional base-2 divisibility offers CashToken issuers greater flexibility in optimizing their denomination choice ("number of decimal places") to their use case; the base-10 component is useful for ergonomics (e.g. representing "cents"), while the base-2 component optimizes "micro-transaction" and high-precision use cases (e.g. fractional cents).

Alternatively, fractional asset encoding could use a radix of 10 to align the resulting fractions with decimal numbers, presumably for ergonomics (most humans think in decimal). However, **optimizing fractional asset encoding for ergonomics would be wasteful and even harmful to end-user privacy**. Humans should not be typing these numbers into software – instead, humans provide numbers only in terms of their preferred units of account. The provided numbers may then passed through one or more exchange rates (if not accounting with BCH or the cashtoken), UTXO selection algorithm(s), privacy-preserving logic ("clean" decimal values leak privacy information), and possibly other functions before being encoded as fractional asset values by the wallet software, typically aiming to minimize the total byte length (and mining fees) of transactions. Even for transaction and blockchain visualization software, rounding and simplifying precise values for human consumption (e.g. USD values) is already common. Optimizing fractional asset encoding for ergonomic decimal representation would waste transaction bandwidth, encourage privacy-harming transaction constructions, and fail to improve real-world user experiences.

### Continued Value of Indivisible CashTokens

Though this proposal introduces fractional CashTokens, indivisible CashTokens continue to be an optimal choice for many fungible token use cases requiring a less-divisible supply, e.g. common stock, voting tokens, event tickets, etc. In these cases, indivisible CashTokens already offer sufficient divisibility, and using fractional cashtokens would generally waste a byte per output (for the unused fraction).

### Existing vs. High-Precision Introspection Opcodes

This proposal avoids impacting the existing behavior of value/amount introspection opcodes, even when inspecting transaction components holding fractional assets. This behavior both 1) ensures backwards compatibility with existing contract systems, and 2) preserves the ability for contracts to optimize operation cost by using the existing, lower-precision introspection operations. While many contract applications may rely exclusively on high-precision operations, lower-precision operations remain particularly useful in cases where calculations must be performed with amounts from many unknown CashToken categories and/or many external UTXOs (e.g. multi-asset market making algorithms).

### Use of Fixed-Precision for High-Precision Introspection Opcodes

The [high-precision introspection opcodes](#high-precision-introspection-opcodes) introduced by this proposal consistently return all values in fixed `1/2^64`th "units". This approach ensures that all calculations behave equivalently between 1) low-precision and high-precision operations and 2) satoshi values and cashtoken amounts.

<!-- ## Implementations

Please see the following implementations for additional examples and test vectors:

- **JavaScript/TypeScript**:
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash. [Branch `next`](https://github.com/bitauth/libauth/tree/next).
  - [Bitauth IDE](https://github.com/bitauth/bitauth-ide) – An online IDE for bitcoin (cash) contracts. [Branch `next`](https://github.com/bitauth/bitauth-ide/tree/next). -->

## Feedback & Reviews

- [TXv5 CHIP Issues](https://github.com/bitjson/bch-txv5/issues)
- [`CHIP 2025-01 TXv5: Transaction Version 5` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/)

## Changelog

This section summarizes the evolution of this document.

- **v1.0.0 – 2025-1-29**
  - Initial publication

## Copyright

This document is placed in the public domain.
