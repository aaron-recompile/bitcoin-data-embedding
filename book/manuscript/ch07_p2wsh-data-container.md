# Chapter 7 P2WSH —— Scripts Become "Legitimate Data Containers"

## P2WSH —— Scripts Become "Legitimate Data Containers"

### From Abusing Structures to Protocol-Native Containers

After the multisig experiments of Counterparty and Stamps, Bitcoin entered the SegWit era.
SegWit introduced a key structure:

**P2WSH (Pay-to-Witness-Script-Hash)**

On the surface, it's the witness version of P2SH,
but in essence, it's the first time the protocol layer provided a "structured script container."

**It should be clarified that:** P2WSH's original design intent was to solve transaction malleability issues and support more complex script logic; the witness discount was a side effect. However, from a data embedding perspective, it objectively provides a "legitimate data container"—scripts themselves can express complex structures, and the witness area provides greater capacity and lower cost for such expression.

If we say:

- **Stamps** are "sneaking data into scripts"

Then **P2WSH** is:

**Openly acknowledging that scripts themselves can carry complex structures, fully exposed in the witness.**

This chapter will explain:

- What structural changes P2WSH actually made
- Why it's naturally suited as a "data container"
- How it differs from OP_RETURN / multisig
- How it paves the way for Ordinals and Atomicals

---

## 7.1 From P2SH to P2WSH: Script Hashing and Script Exposure

### P2SH (Pay-to-Script-Hash)

Recalling P2SH:

```
scriptPubKey: OP_HASH160 <20-byte-hash> OP_EQUAL
```

The script body (redeemScript) is only exposed when spent, provided in scriptSig.

**Characteristics:**
- Script hash: RIPEMD160(SHA256(redeemScript)) = 20 bytes
- Script in scriptSig (affects TXID)
- Data in base transaction (4 weight units per byte)

### P2WSH (Pay-to-Witness-Script-Hash)

SegWit introduced P2WSH:

```
scriptPubKey: OP_0 <32-byte sha256(witnessScript)>
witness:      <... witnessScript and stack ...>
```

**Key Changes:**

1. **Script moved from scriptSig to witness area**
   - No longer affects TXID
   - Solves malleability problem

2. **Script hash changed to sha256, 32 bytes long**
   - More secure hash algorithm
   - Larger hash space

3. **Witness is an independent fee calculation area (weight), more lenient on block capacity**
   - Base transaction: 4 weight units per byte
   - Witness data: 1 weight unit per byte
   - **75% cost discount**

From a data embedding perspective:

**witnessScript = a field that can accommodate arbitrarily complex structures.**

---

## 7.2 Why P2WSH is a "Clean Container" While Stamps is "Abuse"?

The core difference lies in "semantics":

- **Stamps** turn fields that should be public keys into data
  - Semantic misalignment: public keys should be elliptic curve points, but are treated as data
  - Confuses script semantics

- **P2WSH** uses fields that are already scripts for complex logic expression
  - Semantically correct: scripts themselves are meant to express logic
  - Aligns with design intent

### The Nature of Scripts

Scripts are essentially:

**An executable verification program.**

Therefore, using scripts to express complex logic and state commitments is their native function;
it's just that under the combination of P2WSH + witness, this function was realized for the first time with:

- Greater capacity (witness can be large)
- Clearer structure (witness separation)
- More reasonable fee model (weight discount)
- Better engineering controllability (doesn't affect TXID)

---

## 7.3 P2WSH as a "Protocol Container": Typical Patterns

In P2WSH mode, a custom protocol can be constructed like this:

### 1. Using witnessScript to Express Logical Structure

```
witnessScript:
  <protocol_magic> <version> <payload_hash> OP_DROP
  OP_TRUE
```

Where:
- `<protocol_magic>`: Used to identify which protocol
- `<version>`: Version number
- `<payload_hash>`: Hash of off-chain large data (e.g., images, JSON, state machines)
- `OP_DROP`: Discards parts unrelated to verification
- `OP_TRUE`: Minimal logic to ensure script returns true

### 2. Using witness to Pass Runtime Parameters

```
witness: [
  <signature>,
  <pubkey>,
  <witnessScript>,  # Complete script
  <additional_payload>  # Optional: additional data
]
```

### 3. Using sha256(witnessScript) as the Commitment Point in scriptPubKey

```
scriptPubKey: OP_0 <32-byte sha256(witnessScript)>
```

**Functions:**
- Ensures script cannot be modified
- Provides anchor for off-chain state
- Allows selective reveal

From an "embedded data" perspective:

**witnessScript + witness together form a "structured data packet."**

---

## 7.4 P2WSH witnessScript Structure Overview

In P2WSH mode, a typical witnessScript structure can be organized like this:

```
witnessScript:
  <protocol_magic>    # Protocol identifier (variable length)
  <version>           # Version number (1 byte)
  <payload_hash>      # Off-chain data hash (32 bytes)
  <additional_data>   # Optional: additional data (variable length)
  OP_DROP OP_DROP ... # Discard data
  OP_TRUE             # Ensure script passes verification
```

**Key Points:**
- witnessScript can accommodate arbitrarily complex structures
- Committed in scriptPubKey via `sha256(witnessScript)`
- Fully revealed in witness, enjoying 1 WU/byte cost discount

---

## 7.5 Comparison with OP_RETURN and Bare Multisig

| Method | Data Location | Semantic Reasonableness | Capacity | Weight Cost | Typical Use |
|--------|--------------|------------------------|----------|-------------|-------------|
| OP_RETURN | Output | High | 80 bytes (standard policy) | 4 WU/byte | Instructions, short payload |
| Bare Multisig | Output (pubkey) | Low | Medium (33 bytes per pubkey) | 4 WU/byte | Extreme examples like Stamps |
| P2WSH | **Input witness** | **High** | **Very large** | **1 WU/byte** | **Structured protocols, script applications** |

**Summary:**

- **OP_RETURN** is suitable for "short instructions"
- **P2WSH** is suitable for "structured logic and commitments"

The emergence of P2WSH made the path of "scripts as data structures" transform from abuse into a protocol-native possibility.

---

## 7.6 P2WSH Data Location: Commit-Reveal Pattern

**Key Insight:**

P2WSH follows the "commit-reveal" pattern:

- **Output side**: Only stores script hash (32 bytes)
  ```
  scriptPubKey: OP_0 <32-byte sha256(witnessScript)>
  ```
  - Compact, doesn't take up space
  - Provides cryptographic commitment

- **Input side**: Reveals complete script in witness
  ```
  witness: [<signature>, <pubkey>, <witnessScript>]
  ```
  - Data is actually here
  - Enjoys 75% cost discount

This is completely consistent with the "commit-reveal pattern" discussed in Chapter 2.

---

## 7.7 P2WSH Paves the Way for Subsequent Protocols

### 7.7.1 Paving the Way for Ordinals

Ordinals use witness to carry image data, essentially an extension of the P2WSH approach:

- Data in witness (enjoys discount)
- Uses Taproot script path (more structured)
- Doesn't pollute UTXO set (witness can be pruned)

### 7.7.2 Paving the Way for Atomicals

Atomicals use Taproot Merkle trees to store data, also an upgrade of the P2WSH commitment pattern:

- On-chain commitment (Taproot commitment)
- Off-chain data (script leaf in witness)
- Selective reveal

### 7.7.3 Paving the Way for RGB

RGB's client-side validation model is also inspired by P2WSH:

- On-chain commitment (minimal data)
- Off-chain state (complete data)
- Client-side validation (cryptographic proofs)

---

## 7.8 P2WSH's Position in This Book: Bridge Between Script Era and Taproot Era

In the narrative of "Bitcoin's non-transaction data history," P2WSH is a transitional node:

- It didn't directly spawn a hit protocol (unlike Ordinals / RGB)
- But it provided a healthier structure for "embedded data + script logic"
- Paved the way for Taproot script path and Ordinals witness patterns
- Made "scripts as containers" evolve from practice to standard

### Position in Data Embedding History

| Protocol | Data Location | Characteristics | Era |
|----------|--------------|-----------------|-----|
| Counterparty | Output (multisig) | Dual channels, occupies UTXO | Legacy |
| Stamp | Output (multisig) | Extreme, UTXO pollution | Legacy |
| **P2WSH** | **Input witness** | **Structured, enjoys discount** | **SegWit Era** |
| Ordinals | Input witness | Large capacity, image storage | Taproot Era |
| Atomicals | Input witness + Taproot | Merkle-ized, structured | Taproot Era |

---

## 7.9 Summary: P2WSH's Engineering Significance

P2WSH represents:

- ✔ "Legitimization" of scripts as data containers
- ✔ First mature application of commit-reveal pattern
- ✔ Economic advantage of witness discount
- ✔ Foundation for subsequent protocols (Ordinals, Atomicals, RGB)
- ✔ Turning point from "abusing structures" to "protocol-native"

And points the direction for future development:

**Data should be placed in witness, enjoying cost discounts.**
**Scripts can serve as structured data containers.**

This leads to the next chapter:

**Chapter 8: Ordinals —— Using Witness to Turn sats into "Carriers"**

The data explosion of the witness era, where Ordinals pushed the P2WSH approach to the extreme, directly storing images and content in witness.

