# Chapter 8 —— Ordinals —— Taproot Envelopes and the Data Layer Explosion

## From Witness Containers to Social Phenomenon

In the previous chapter, we saw that P2WSH moved scripts from scriptSig to witness,
bringing a 75% cost discount and clearer structural separation.

But what truly ignited "witness as a data layer" was the **Taproot upgrade** activated in November 2021.

Taproot (BIP 342 Tapscript) did one crucial thing:

**Removed the 520-byte limit on individual witness stack elements and the 10,000-byte limit on total script size.**

In SegWit-era P2WSH, although data was already in witness, it still followed traditional script engine rules,
with the 520-byte limit on individual witness stack elements remaining in effect.
After Taproot, as long as the entire block does not exceed 4MB (approximately 4 million weight units),
a single transaction's witness can theoretically fill the entire extended block.

This opened the door to "stuffing arbitrary large data into Bitcoin."

In January 2023, Casey Rodarmor released the **Ordinals protocol**.
It wasn't the first protocol to utilize witness,
but it was the first to push this capability **to the social level, igniting controversy**.

This chapter will analyze:

- Ordinals' two-step design: satoshi ordering + content inscription
- commit-reveal pattern and OP_FALSE OP_IF envelopes
- Why Taproot script path is an ideal data container
- Engineering reproduction: constructing an Ordinals-style inscription
- Controversy and boundaries
- End of chapter: brief introduction to variant protocols like Atomicals

---

## 8.1 Ordinals' First Step: Numbering Every Satoshi

Ordinals first does one thing, without discussing data:

**Establishes a definitive ordering and numbering rule for all satoshis.**

Core rule: **First In, First Out (FIFO)**

```
Block 1 → Block 2 → Block 3 → ...
  ↓        ↓        ↓
Tx 1 → Tx 2 → Tx 3 → ...
  ↓        ↓        ↓
Input → Output → Input → Output → ...
  ↓        ↓
Split by amount into sats, number sequentially
```

In this way, theoretically every sat can be assigned an "ordinal number,"
ranging from 0 to 2,100,000,000,000,000 (2.1 quadrillion).

For example:

> "This sat's ordinal number is 1,234,567,890, first appeared in block 100,000's coinbase,
> later went through addresses A → B → C."

**Key Insight:**

This step is entirely completed at the **interpretation layer**, not involving consensus.
The Bitcoin protocol itself doesn't know about "ordinal number" existence;
this is just an external observer's tracking rule for sat flow.

---

## 8.2 Second Step: "Inscribing" Data in Witness

Ordinals' second step:

**Write a piece of data in the witness of a transaction carrying a certain sat, defined as an "inscription."**

### 8.2.1 commit-reveal Two-Phase Pattern

Inscriptions use a **two-phase** process:

**Phase One: Commit (Commitment)**

Create a Taproot output (P2TR) that internally commits to a script containing inscription data.

```
scriptPubKey: OP_1 <32-byte tweaked pubkey>
```

At this point, the inscription content is not yet exposed on-chain.
External observers can only see an ordinary Taproot address.

**Phase Two: Reveal**

Spend the above Taproot output, revealing the inscription content through script path.

```
witness:
  <signature>
  <inscription script>
  <control block>
```

The inscription data is in `<inscription script>`, and only now is it truly "on-chain."

### 8.2.2 Why Use commit-reveal?

1. **Prevent front-running**: If inscription content is directly exposed in a single transaction,
   miners or other observers may copy the same content first. commit-reveal ensures content is invisible before confirmation.

   **Timing details**: The commit transaction must be confirmed first before the reveal transaction can be safely broadcast.
   If both transactions are broadcast simultaneously, the reveal content is visible in the mempool,
   and miners or monitors can preemptively construct their own commit committing the same content,
   front-running by reordering transactions or paying higher fees. Most inscription tools default to waiting for commit confirmation before broadcasting reveal.

2. **Optimize fees**: The commit phase output is only a 32-byte hash,
   large data appears in the reveal phase, enjoying witness discount at that time.

3. **Aligns with Taproot design**: Taproot script path itself is a "commit-reveal" structure.

---

## 8.3 OP_FALSE OP_IF Envelope: Making Data "Non-Executable"

Inscription data is wrapped in a special "envelope" structure:

```
OP_FALSE
OP_IF
  OP_PUSH "ord"
  OP_PUSH 1
  OP_PUSH <content-type>
  OP_PUSH 0
  OP_PUSH <data chunk 1>
  OP_PUSH <data chunk 2>
  ...
OP_ENDIF
```

### 8.3.1 Why This Design?

**OP_FALSE OP_IF ... OP_ENDIF** function:

1. `OP_FALSE` pushes 0 onto the stack
2. `OP_IF` checks the stack top, finds it's 0 (false), skips the entire IF block
3. PUSHDATA instructions within the block **never execute**
4. `OP_ENDIF` ends the conditional block
5. Script continues executing subsequent verification logic (usually `<pubkey> OP_CHECKSIG`)

**Result:**

- Data is "stuffed" into the script but doesn't participate in execution
- Doesn't occupy stack space
- Doesn't affect script verification results
- From Bitcoin nodes' perspective, this is "legal but useless" script data

### 8.3.2 Complete Inscription Script Example

A typical inscription script:

```
OP_FALSE
OP_IF
  OP_PUSH "ord"           # Protocol identifier
  OP_PUSH 1               # Indicates next push is content-type
  OP_PUSH "text/plain"    # MIME type
  OP_PUSH 0               # Indicates content follows
  OP_PUSH "Hello, Ordinals!"
OP_ENDIF
<x-only pubkey>
OP_CHECKSIG
```

Execution flow:

1. `OP_FALSE` pushes 0 onto the stack
2. `OP_IF` checks stack top is 0 (false), skips the entire IF block
3. All PUSHDATA instructions within the IF block **never execute**, but data remains in script bytes
4. `<pubkey> OP_CHECKSIG` verifies signature provided by witness → returns TRUE
5. Stack top is TRUE, script passes

**Key point: Data remains in script bytes but never participates in computation.**

---

## 8.4 Taproot Script Path: Ideal Data Container

### 8.4.1 Why Taproot Instead of P2WSH?

| Feature | P2WSH | Taproot (P2TR) |
|---------|-------|----------------|
| Witness element size limit | 520 bytes | **No single element size limit** |
| Script size limit | 10,000 bytes | **No script size limit** (only block limit) |
| Structure | Single witnessScript | **Merkle tree** (multiple leaves) |
| Privacy | Reveals complete script | Only reveals used leaf |
| Cost | 1 WU/byte | 1 WU/byte |

**Taproot's Key Breakthrough:**

BIP 342 (Tapscript) removed:
- 520-byte limit on individual pushes
- 10,000-byte limit on total script size

Now, as long as the entire block's weight doesn't exceed 4,000,000 WU (approximately 4MB),
a single transaction's witness can be arbitrarily large.

**Technical Note:**
Although Ordinals could theoretically be implemented on P2WSH (subject to 520-byte element limit),
all actual implementations choose Taproot because Taproot removed size limits,
making it more suitable for storing large files (images, videos, etc.).

### 8.4.2 Taproot Internal Structure Review

```
                    [Tweaked Public Key]
                           |
                    [Internal Key] + [Merkle Root]
                                          |
                                   ┌──────┴──────┐
                                [Leaf A]      [Leaf B]
                                   |
                         OP_FALSE OP_IF
                           <inscription>
                         OP_ENDIF
                         <pubkey> OP_CHECKSIG
```

Inscription data is placed in a leaf script,
revealed when spent through control block and Merkle proof.

---

## 8.5 Engineering Reproduction: Constructing a Simplified Inscription

Core process for constructing Ordinals-style inscriptions:

**Phase One: Commit (Commitment)**

1. Generate key pair, construct inscription script (including OP_FALSE OP_IF envelope)
2. Place script into Taproot leaf, calculate Merkle root
3. Calculate tweaked pubkey using internal pubkey and Merkle root
4. Generate P2TR address, send BTC to that address

At this point, inscription content is not yet exposed; externally only an ordinary Taproot output is visible.

**Phase Two: Reveal (Revelation)**

1. Spend the above Taproot output
2. Provide in witness:
   - signature (signature for the transaction)
   - inscription_script (complete inscription script)
   - control_block (Merkle proof)
3. Broadcast transaction, inscription content officially goes on-chain

**Parsing Inscriptions**

Extract inscription script from on-chain transaction witness, parse data within OP_FALSE OP_IF block:
- Identify "ord" protocol marker
- Extract content-type and content

Through this process, readers can understand:

**Inscriptions are not magic, just bytes in witness + an interpreter.**

> **Note:** Complete runnable code examples are available in the companion code repository `code/ordinals/` directory.

---

## 8.6 Data Flow and UTXO Relationship

### 8.6.1 Inscriptions Don't Pollute UTXO Set

This is Ordinals' key advantage over Stamps:

| Method | Data Location | Impact on UTXO Set |
|--------|---------------|-------------------|
| Stamps (bare multisig) | Output scriptPubKey | **Permanently occupies** UTXO set |
| Ordinals | Input witness | **No impact** on UTXO set |

**Reason:**

- Stamps place data in **output** scriptPubKey (fake pubkeys),
  these outputs cannot be spent, permanently remaining in the UTXO set.

- Ordinals place data in **input** witness,
  once the transaction is confirmed, witness data **is stored in block history** (truly on-chain),
  but doesn't enter the UTXO set.

### 8.6.2 Data Storage Location

```
Block Structure:
├── Block Header
├── Transactions
│   ├── Tx 1
│   │   ├── Inputs
│   │   │   ├── prevout
│   │   │   └── witness  ← Inscription data here
│   │   └── Outputs
│   └── Tx 2 ...
└── ...

UTXO Set:
├── Output A (spendable)
├── Output B (spendable)
└── ...  ← Does not contain witness data
```

**Full Node Operating Costs:**

- UTXO set needs to reside in memory/SSD, size-sensitive
- Block history can be stored on mechanical hard drives, size less sensitive
- Ordinals increases block history size but doesn't add UTXO set burden

**Important Clarification:**
Inscription data is indeed stored on-chain (in block history), accessible to any node with complete block history.
The difference is: UTXO set needs to reside in memory/SSD to support fast queries, while block history can be stored on mechanical hard drives,
with less impact on node performance. This doesn't mean data is "not on-chain," just different storage locations and access methods.

### 8.6.3 Inscription "Persistence" Issues

**Pruned Node Challenges:**

Users running pruned nodes (only keeping the most recent N blocks) may not be able to access historical inscription data.
This is because:

- Pruned nodes delete old block data to save space
- Inscription data is stored in block history witness
- Once blocks are deleted, inscription data becomes inaccessible

**Data Guarantee Levels:**

| Node Type | Can Access Historical Inscriptions | Data Guarantee |
|-----------|-----------------------------------|----------------|
| Full node | ✅ Yes | Complete historical data |
| Pruned node | ❌ No (old blocks deleted) | Only keeps most recent N blocks |
| Light node | Depends on indexer | Depends on third-party services |

**Key Insight:**

Ordinals' "data on-chain" depends on **full nodes retaining complete block history**.
For pruned node users, inscription data may be inaccessible, exposing witness data storage "persistence" issues:
Data is indeed on-chain, but access requires complete historical data.

**Extended Impact of assumevalid:**

Even for archival nodes (retaining complete block history), when syncing with the `assumevalid` parameter,
script execution and signature verification for historical blocks are skipped.
This means nodes download and store witness data but don't personally verify its validity.
Inscription data's "accessibility" and "verified validity" are two different levels of guarantee:
- **Accessibility**: Node stores data, can read it
- **Verified validity**: Node personally executed script verification, confirmed data validity

For nodes relying on `assumevalid` sync, inscription data is accessible but verification relies on social consensus rather than local validation.

---

## 8.7 Controversy: Innovation or Spam?

Ordinals sparked one of Bitcoin community's most intense debates.

### 8.7.1 Proponents' Views

1. **Protocol allows it**: Bitcoin scripts inherently allow arbitrary data, this is freedom of use
2. **Pay to use**: Inscription users pay real fees for block space
3. **Witness pricing**: Witness data enjoys 75% discount, this is protocol design result
4. **Increased demand**: Inscriptions bring new block space demand, increasing miner revenue
5. **Extended use cases**: Demonstrates Bitcoin's potential as a data layer

### 8.7.2 Opponents' Views

1. **Deviates from original purpose**: Bitcoin is a "peer-to-peer electronic cash system," not an image hosting service
2. **Drives up fees**: Large-scale inscriptions cause ordinary transaction fees to soar
3. **Node burden**: Block bloat increases sync and storage costs
4. **Spam data**: Large amounts of inscription content are low-quality speculative NFTs
5. **Policy tightening**: May push Core to tighten standard rules, affecting other use cases

### 8.7.3 Core Developers' Response

Some Core developers proposed tightening policy rules like `datacarriersize`,
but these are only nodes' **relay policies**, not consensus rules.
Miners can still package any consensus-compliant transactions.

Luke Dashjr submitted a PR attempting to filter inscription transactions, but it wasn't merged.
The rough consensus the community reached is:

> "Don't like it, but can't stop it. Protocol layer shouldn't censor legal transactions."

### 8.7.4 This Book's Position

This book doesn't take sides, only analyzes from structural and engineering perspectives:

- Ordinals **is technically legal**: It uses only standard opcodes and structures
- It **exposes witness discount design consequences**: Cheap storage space will be utilized
- It **sparks discussions about Bitcoin's boundaries**: Monetary system vs. general-purpose data layer

These discussions are valuable for understanding Bitcoin's essence.

---

## 8.8 Ordinals' Position in Data Embedding History

| Era | Protocol | Data Location | Characteristics |
|-----|----------|---------------|-----------------|
| Genesis | Coinbase | Coinbase | Only miners can use |
| OP_RETURN | Various protocols | Output | 80-byte limit |
| Multisig abuse | Counterparty, Stamps | Output (pseudo-pubkeys) | Pollutes UTXO |
| SegWit | P2WSH | Witness | 520-byte element limit |
| **Taproot** | **Ordinals** | **Witness** | **No size limit, social explosion** |

Ordinals marks:

1. **The formal explosion of witness-era data embedding**
2. **From protocol engineers' toys to ordinary users' speculation objects**
3. **script/witness entering public consciousness for the first time**

---

## 8.9 Variant Protocols: Atomicals, BRC-20, and Others

Ordinals opened the door, spawning a series of variant protocols.
They share the same underlying mechanisms but differ in upper-layer design focus.

### 8.9.1 Atomicals: Satoshi Coloring (Colored Coins) + Structured Objects

**Release Date**: September 2023

**Core Differences:**

| Dimension | Ordinals | Atomicals |
|-----------|----------|-----------|
| Envelope marker | `"ord"` | `"atom"` |
| Satoshi handling | Number sats (ordinal number) | "Color" sats (colored coin) |
| Data model | Content (images/text) | Structured objects (Asset/NFT/Realm) |
| Token standard | BRC-20 (JSON inscription) | ARC-20 (1:1 sat-pegged) |
| Additional features | None | **Bitwork mining** (optional PoW) |

**Bitwork Mining** (Engineering optimization highlight):

Atomicals introduced an optional PoW mechanism requiring minting transaction TXIDs to start with a specific prefix.
By adjusting nonce fields in witness, continuously recalculating transaction hashes until requirements are met.
This increases minting costs, prevents spam, and provides objective measurement for "rarity."

> **Note:** Specific implementation details and engineering design rationale for Bitwork mining are detailed in the appendix or supplementary chapters.

**ARC-20 vs BRC-20:**

| Dimension | BRC-20 | ARC-20 |
|-----------|--------|--------|
| Data format | JSON inscription | CBOR encoding |
| Token unit | Arbitrary (JSON defined) | 1 token = 1 sat |
| Transfer logic | Off-chain indexer parses JSON | **Native UTXO rules** |

ARC-20's "1 token = 1 sat" design makes token transfers directly follow Bitcoin's UTXO rules,
without complex off-chain indexing.

**Tradeoff Analysis:**

This design returns to early colored coin thinking, using satoshi itself as the token carrier.

**Benefits:**
- State is UTXO, transfer is spending, no need to maintain off-chain state database
- Different indexers won't produce balance ambiguities (state is on-chain, parsing is unique)
- Highly consistent with Bitcoin's native model
- Transfers can be done with ordinary Bitcoin wallets supporting coin control, no specialized ARC-20 wallet required

**Costs:**
- Token minimum unit locked to 1 sat, cannot be subdivided
- Total supply limited by colored sat count, cannot exceed Bitcoin's total supply
- When BTC price is high, transferring small amounts of tokens becomes uneconomical (since real satoshis must be moved as carriers)

### 8.9.2 BRC-20: JSON Inscription Tokens

**Release Date**: March 2023

BRC-20 uses Ordinals inscriptions of JSON to define tokens:

```json
{
  "p": "brc-20",
  "op": "deploy",
  "tick": "ordi",
  "max": "21000000",
  "lim": "1000"
}
```

**Problems:**

- Token state completely depends on off-chain indexer parsing
- Different indexers may produce different results
- Disconnected from Bitcoin's native UTXO model

### 8.9.3 Runes: Casey Rodarmor's "Correction"

**Release Date**: April 2024 (Bitcoin's fourth halving)

Ordinals creator Casey Rodarmor later launched the **Runes** protocol,
attempting to implement token functionality in a simpler way:

- Returns to OP_RETURN (instead of witness)
- Token state directly bound to UTXO
- No complex indexer needed

This is a response to BRC-20 chaos, detailed in the next chapter.

### 8.9.4 Variant Protocol Comparison Summary

| Protocol | Data Location | Envelope/Format | Core Innovation |
|----------|---------------|-----------------|-----------------|
| Ordinals | Witness | `"ord"` + MIME | Sat numbering + content inscription |
| Atomicals | Witness | `"atom"` + CBOR | Sat coloring + Bitwork mining |
| BRC-20 | Witness | JSON | Off-chain indexed tokens |
| Runes | OP_RETURN | Binary | UTXO-native tokens |

**Common Points:**

- All utilize post-Taproot witness capacity
- All adopt commit-reveal pattern
- All depend on off-chain interpreters to assign "meaning" to data

**Key Insight:**

> These protocols' underlying mechanisms are almost identical;
> differences lie in upper-layer **data models** and **social narratives**.

---

## 8.10 Summary: Taproot Envelope Paradigm

Ordinals and its variants demonstrate a universal paradigm:

```
┌─────────────────────────────────────────────┐
│        Taproot Envelope Paradigm             │
├─────────────────────────────────────────────┤
│  1. Commit: Create P2TR output, commit script hash │
│  2. Reveal: Spend output, reveal script path      │
│  3. Envelope: OP_FALSE OP_IF <data> OP_ENDIF       │
│  4. Interpretation: Off-chain indexer parses data, assigns semantics │
└─────────────────────────────────────────────┘
```

**Engineering Significance:**

- Taproot provides structure (Merkle tree, script path)
- OP_FALSE OP_IF provides "non-executable data container"
- commit-reveal provides privacy and front-running prevention
- Witness discount provides economic incentive

**Social Significance:**

- First time ordinary users realized Bitcoin can store arbitrary data
- Sparked deep discussions about Bitcoin's essence
- Provided comparative reference for subsequent protocols (RGB, Runes, etc.)

Next chapter, we'll see a "return" attempt:

**Chapter 9: Runes —— Redefining Tokens with OP_RETURN**

How Casey Rodarmor attempts to correct BRC-20 chaos with a simpler design.

