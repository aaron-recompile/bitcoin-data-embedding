# Chapter 2 — The Complete Map of Bitcoin Data Entry Points

## The Complete Map of Bitcoin Data Entry Points: Input vs Output Structure

### The Complete Map of Where Data Can and Cannot Enter Bitcoin

The previous chapter established a core fact:

Bitcoin preserves only two types of bytes: input data (for unlocking) and output data (for locking).
All non-transaction data must be written along these two directions.

However, this statement remains too abstract.
This chapter will systematically deconstruct the actual Bitcoin transaction structure, clarifying:
which fields can carry bytes, how many, what rules apply, which are consensus vs policy, which can be abused, and which cannot.

**More importantly, we need to understand a crucial distinction:**

- **Data in Output**: Data is permanently visible upon creation, directly stored in the UTXO set
- **Data in Input Witness**: Data is revealed only when unlocking, carried through the witness field

This distinction determines the economic cost, visibility, and protocol design choices for data.

This chapter can serve directly as a "complete data embedding map" to help understand the underlying technical foundations of all protocols.

---

## 2.1 Overview of Bitcoin Transaction Byte Structure

### 2.1.1 The Four Major Components of a Transaction

A Bitcoin transaction consists of four major blocks:

```
Version
Inputs[]
Outputs[]
Locktime
```

**Only Inputs and Outputs contain space for "insertable bytes".**

However, it is important to clarify: **the actual storage location of data** may differ from **the entry point for creating data structures**.

### 2.1.2 Transaction Structure Visualization

Before diving into each data entry point, let us establish a complete picture of the transaction structure.

**Inputs Structure:**

```
┌─────────────────────────────────────────┐
│ Input #0                                │
│ ├─ Previous TXID (32 bytes)            │
│ ├─ Previous Vout (4 bytes)             │
│ ├─ scriptSig Length (VarInt)           │
│ ├─ scriptSig (📝 Data embeddable - Legacy)  │
│ ├─ sequence (4 bytes)                  │
│ └─ witness (📝 Data embeddable - SegWit)    │
│    └─ [element_0, element_1, ...]      │
│                                         │
│ Input #1                                │
│ └─ ...                                  │
└─────────────────────────────────────────┘
```

**Outputs Structure:**

```
┌─────────────────────────────────────────┐
│ Output #0                               │
│ ├─ Value (8 bytes)                     │
│ └─ scriptPubKey Length (VarInt)        │
│    └─ scriptPubKey                     │
│       ├─ OP_RETURN <data> (📝 Method 1)   │
│       ├─ Fake Multisig (📝 Method 2)     │
│       └─ Special script structure (📝 Method 3)      │
│                                         │
│ Output #1                               │
│ └─ ...                                  │
└─────────────────────────────────────────┘
```

### 2.1.3 Data Entry Point Identification

**Key Markers:**
- 📝 Indicates locations where arbitrary data can be embedded
- Legacy transactions use scriptSig; SegWit/Taproot transactions use witness
- Output data entry points: OP_RETURN, Fake Multisig, special scriptPubKey

**Key Distinction:**
- **Data in Output**: Data is permanently visible upon creation, directly stored in the UTXO set
- **Data in Input Witness**: Data is revealed only when unlocking, carried through the witness field, enjoying a 75% cost discount

The following sections will analyze these data entry points one by one.

---

## 2.2 Input Data Entry Points

A simplified Input structure is as follows:

```
{
  prevout: <txid:vout_index>
  scriptSig: <arbitrary bytes>   # legacy
  sequence: <4 bytes>
  witness: [stack elements]      # segwit
}
```

**There are two fields within Input that can write data:**

1. **scriptSig** (legacy)
2. **witness** (segwit)

Their capabilities differ dramatically.

### 2.2.1 scriptSig — The Legacy "Insertable Bytes" Entry Point (Largely Deprecated)

scriptSig is essentially:

```
<push> <signature>
<push> <public key>
```

Beyond legal requirements, it once could hold arbitrary bytes.
This led to early practices such as:

- Stuffing data as "fake scripts"
- Placing private data (extremely wasteful)
- Encoding state through spending paths (Colored Coins pattern)

However:

- ❌ After 2017, this channel was largely abandoned
  - scriptSig affects TXID
  - Prone to introducing malleability
  - Mempool policy discourages using scripts as data containers

Therefore: scriptSig is a "historical embedding path," but remains one of the data entry points.

### 2.2.1 scriptSig Example

Early Colored Coins protocols (such as Colored Coins) embedded data in scriptSig.

**Historical Background:**
- Between 2012-2014, some Colored Coins implementations attempted to encode asset information in scriptSig
- Since scriptSig affects TXID, this approach easily leads to transaction malleability issues
- After SegWit activation in 2017, scriptSig data embedding was largely abandoned

**Current Status:**
- scriptSig is rarely used as a data carrier today
- Modern protocols primarily use witness or OP_RETURN
- See Chapter 3 for the complete history of Colored Coins

### 2.2.2 witness — The Most Powerful Data Carrier in Bitcoin History

After witness emerged, data embedding entered the "modern era."

Witness structure:

```
witness: [ element_0, element_1, ..., element_n ]
```

Each witness element (stack item) is:

- Arbitrary byte length (limited by block weight)
- Does not affect TXID
- Does not enter script execution logic (only pushed)
- Must conform to stack push rules
- Need not have logical meaning (as long as the script validates)

**Witness Discount Mechanism (BIP 141):**

- Base transaction data: 4 weight units per byte
- Witness data: 1 weight unit per byte
- Block weight limit: 4,000,000 units (approximately 4MB)

This means:

- Witness data enjoys a **75% cost discount** compared to traditional data
- Theoretically, nearly 4MB of witness data can be embedded in a single block
- This is the economic reason why Ordinals and Atomicals choose witness

This is extremely important:

**Each element in witness need not be "meaningful"; it only needs to be pushed onto the stack and then ignored during script execution.**

This provides tremendous freedom for:

- Ordinals (images)
- BRC-20 (structured JSON)
- Atomicals (payload)
- Taproot script-path embedding
- Any type of binary data

Witness limitations primarily come from two sources:

1. Block weight (soft limit but important)
2. Script must execute successfully (but can be designed to always succeed)

This is why Ordinals and Atomicals can embed several MB of content.

**Witness is the true "modern data entry point."**

### 2.2.3 Practical Case: Ordinals Inscription Transaction

**Transaction:** `6fb976ab49dcec017f1e201e84395983204ae1a7c2abf7ced0a85d692e442799`

[Mempool Link](https://mempool.space/tx/6fb976ab49dcec017f1e201e84395983204ae1a7c2abf7ced0a85d692e442799)

This is a typical Ordinals inscription transaction that inscribes a PNG image on Bitcoin.

**Witness Structure:**

```
[0] Signature: 152b336f7b6fc69be82df72bb4653eed...

[1] Script (Tapscript):
    - OP_PUSH <internal pubkey>
    - OP_CHECKSIG
    - OP_FALSE OP_IF
      OP_PUSH "ord"
      OP_PUSH 0x01
      OP_PUSH "image/png"
      OP_0
      OP_PUSHDATA2 [Complete PNG data ~2KB]
      OP_ENDIF

[2] Control Block: c04a3ca2cf35f7902df1215f...
```

**Key Points:**

- Data is wrapped in an `OP_FALSE OP_IF ... OP_ENDIF` envelope
- This code is never executed (because of OP_FALSE)
- But the data is permanently stored on the blockchain
- Enjoys the 75% cost discount of witness

---

## 2.3 Output Data Entry Points: Two Fundamental Modes

Output structure:

```
{
  value: <8 bytes>
  scriptPubKey: <locking script>
}
```

**Key Understanding:** All Output data is stored in the `scriptPubKey` field, but there are two fundamentally different modes.

### Mode A: Direct Data Storage (Data-in-ScriptPubKey)

Data is directly exposed in scriptPubKey, permanently visible upon creation, stored in the UTXO set.

**Includes three methods:**

1. **OP_RETURN** - scriptPubKey content is `OP_RETURN <data>` (specifically for data embedding)
2. **Bare Multisig** - Data encoded as "fake public keys" placed in multisig scripts
3. **scriptPubKey Body** - Data disguised as executable script structures (e.g., `<push data> OP_DROP ...`)

**Characteristics:**
- Data is permanently visible upon creation
- Directly stored in the UTXO set
- No cost discount (4 WU/byte)

### Mode B: Commitment Storage (Commitment-in-ScriptPubKey)

scriptPubKey contains only a commitment (hash or tweaked pubkey); actual data is revealed in subsequent Inputs.

**Includes two methods:**

1. **P2SH/P2WSH** - Hash commitment (see Section 2.4)
2. **Taproot** - Merkle root commitment (see Section 2.4)

**Characteristics:**
- Output remains small (only commitment)
- Data revealed in Input witness/scriptSig
- Enjoys witness discount (1 WU/byte)

---

**This section (2.3) details Mode A (direct data storage); Mode B (commitment storage) is covered in Section 2.4.**

### 2.3.1 OP_RETURN — The Most Direct Data Entry Point ("Boldly Writing Data")

Script:

```
OP_RETURN <data>
```

**Data Location: Directly in Output's scriptPubKey**

Characteristics:

- Unspendable (consensus)
- Nodes need not validate any logic
- Each output is independent
- Data interpreted by external protocols
- **Data is permanently visible upon creation**

Limitations (pre-v30):

- Block policy limits size (80 bytes → configurable)
- Single output cannot be too large (to avoid spam)
- Maximum 1 OP_RETURN output per transaction

Limitations (post-v30):

- Block policy limit increased to 100,000 bytes (actually constrained by transaction size and block weight)
- Multiple OP_RETURN outputs allowed
- Configurable via `-datacarriersize` parameter

**Evolution of OP_RETURN Size Limits:**

- **2014 (Bitcoin Core 0.9)**: 40 bytes
- **2015 (Bitcoin Core 0.11)**: 80 bytes
- **2017 to present (v0.11-v29)**: Default 80 bytes (configurable at policy layer)
- **October 2025 (Bitcoin Core v30)**: 100,000 bytes (actually constrained by transaction size and block weight)
  - First time allowing multiple OP_RETURN outputs
  - Configurable via `-datacarriersize` parameter
  - This change sparked controversy in the community; implementations like Bitcoin Knots chose to maintain the 80-byte limit

In practice, the consensus layer has no hard limit on OP_RETURN size, but:

- Pre-v30: Mempool policy defaults to rejecting OP_RETURN exceeding 80 bytes
- Post-v30: Policy layer limit significantly relaxed to 100,000 bytes, with multiple OP_RETURN outputs allowed
- Miners can adjust acceptance standards (via `-datacarriersize` parameter)

OP_RETURN is:

- ✔ The clearest data entry point
- ✔ Also the entry point most guarded by Core developers
- ✔ The foundation for Omni, Counterparty (later), and Runes

### 2.3.1 OP_RETURN Example - USDT (Omni Layer)

**Transaction:** `21c3354ab4422faca43761826884bb4398871a2c404a2fff71be91c9502d3ba0`

[Mempool Link](https://mempool.space/tx/21c3354ab4422faca43761826884bb4398871a2c404a2fff71be91c9502d3ba0)

This is a typical USDT transfer transaction (based on the Omni protocol).

**Output #1 (OP_RETURN):**

```
scriptPubKey hex:
6a146f6d6e69000000000000001f000000003b9aca00

Parsing:
6a          = OP_RETURN
14          = PUSH 20 bytes
6f6d6e69    = "omni" (ASCII)
00000000    = Transaction version
0000001f    = Transaction type (Simple Send)
0000001f    = Property ID 31 (USDT)
000000003b9aca00 = Amount (10.00000000 USDT)
```

**Key Points:**

- OP_RETURN output value is 0
- Maximum 80 bytes of data (pre-v30)
- Protocol parsers read these bytes to understand transfer intent
- This is the standard format for the Omni Layer protocol

### 2.3.2 Bare Multisig — Stuffing Data as "Fake Public Keys"

For example:

```
OP_1 <fake_pubkey> <fake_pubkey> <fake_pubkey> OP_3 OP_CHECKMULTISIG
```

A public key is 33 bytes
A multisig can hold N public keys
These public keys can be arbitrary bytes
→ **This is exactly what the Stamp protocol does.**

**Data Location: Directly in Output's scriptPubKey (fake public keys)**

This path is extremely powerful but also extremely dangerous.

Consensus allows it; policy initially allowed it, later discouraged it.

**Why did Stamps choose this approach?**

- Wanted data permanently in output (UTXO set)
- Did not want to rely on witness (witness was not yet widespread when early Stamps were designed)
- Data visible upon creation, no unlocking needed

### 2.3.3 Fake Multisig Example - Stamp Protocol

**Transaction:** `b1278acf50c27342753d01af9013a709509fdd920bc40ddcbec0738aaec2764c`

[Mempool Link](https://mempool.space/tx/b1278acf50c27342753d01af9013a709509fdd920bc40ddcbec0738aaec2764c)

The Stamp protocol uses "fake public keys" to embed data in UTXOs.

**Characteristics:**

- Creates multiple small outputs (typically 330-546 sats)
- Each output's scriptPubKey appears like a normal address
- But the public keys actually contain encoded image data
- Data permanently exists in the UTXO set

**Example output addresses:**

```
bc1qapfywj2x8qukztqpcgqlxqqqqqqqqqrxqqq9zhp84vqqpq5t8lyqfvkk5y
bc1q5m4ywq8lcgq0hlxylh7axqqqqqqqqqqqqqqqqqqqqqqqqqqqqqsskk5xx9
```

**Note:** The large number of "q" characters in the addresses—these are encoded data.

The Stamp protocol encodes image data as multiple fake public keys, each 33 bytes, distributed across multiple outputs to ensure data permanently exists in the UTXO set.

### 2.3.3 scriptPubKey Body — "Data Disguised as Script"

Any script can exist as long as it executes correctly.

Therefore, one can construct:

```
<push data> OP_DROP <standard script>
```

Or:

```
<push big data> OP_IF ... OP_ENDIF
```

These are called "poison scripts" or "big data scripts."

**Data Location: Directly in Output's scriptPubKey**

Theoretically, you can stuff any data, as long as the script evaluates to true.

However:

- ❌ Mempool policy will reject non-standard scripts
- ❌ Will create permanently unspendable UTXOs (polluting the UTXO set)

Therefore, this is not a mainstream data embedding method.

---

## 2.4 Commit-Reveal Pattern: Output Locks, Input Reveals

This is the most important yet most misunderstood pattern in Bitcoin data embedding.

**Core Principle:**

- **Output Side**: Stores only a commitment (hash or tweaked pubkey); data is not here
- **Input Side**: Reveals actual data in witness
- **Actual Data Location**: In Input's witness

Advantages of this pattern:

- Data enjoys the 75% cost discount of witness
- Output remains small (only commitment)
- Data revealed only when unlocking (selective reveal possible)

### 2.4.1 P2SH / P2WSH — Hash Commitment Pattern

**P2SH (Pay to Script Hash):**

Output Side (creating lock):

```
scriptPubKey: OP_HASH160 <20-byte-hash> OP_EQUAL
```

Here there is only a hash commitment, **no actual data**.

Input Side (when unlocking):

```
scriptSig: <redeemScript>   # legacy
```

Or (SegWit version):

```
witness: [
  <signature>,
  <pubkey>,
  <witnessScript>  ← Actual data is here!
]
```

**Actual Data Location: In Input's witness (witnessScript)**

**P2WSH (Pay to Witness Script Hash):**

Output Side (creating lock):

```
scriptPubKey: OP_0 <32-byte-sha256(witnessScript)>
```

Here there is only a hash commitment, **no actual data**.

Input Side (when unlocking):

```
witness: [
  <signature>,
  <pubkey>,
  <witnessScript>  ← Actual data is here!
]
```

**Actual Data Location: In Input's witness (witnessScript)**

**Key Insight:**

- P2WSH places data in **input's witness**, not output!
- Output is just a "hash lock"
- The actual witnessScript is revealed in witness when unlocking
- This is why P2WSH can carry large data—because data is in witness, enjoying a 75% discount

Both can turn scripts into:

- Custom data structures
- State machines
- Metadata carriers
- Sub-protocol carriers
- Merkle leaf containers

P2WSH is a true "data container," but the data is in witness.

### 2.4.2 Taproot (P2TR) — Merkle Commitment Pattern

**Taproot's script path:**

Output Side (creating lock):

```
scriptPubKey: OP_1 <32-byte-tweaked-pubkey>
```

Here there is only a tweaked public key, containing a Merkle root commitment, **no actual data**.

Input Side (when unlocking via script path):

```
witness: [
  <signature/data>,
  <script leaf>,       ← Actual script is here!
  <control block>      ← Merkle proof is here!
]
```

**Actual Data Location: In Input's witness (script leaf + control block)**

**Key Insight:**

- Taproot also places data in **input's witness**!
- Output is just a "commitment" (tweaked pubkey)
- Script leaf and Merkle path are revealed in witness when unlocking

You can construct:

- Massive Merkle trees
- Each leaf storing structured data
- Selective reveal using control block
- Encoding complex protocol states

Ordinals, Atomicals, RGB, and Taproot Assets all rely on it.

**Why do Ordinals use Taproot instead of OP_RETURN?**

- Taproot's data is in witness (75% discount)
- OP_RETURN's data is in output (no discount)
- Taproot can carry larger data

**Why does RGB use Taproot commitment?**

- Output side only stores hash commitment (small)
- Data is off-chain (not on-chain)
- Only minimal commitment stored on-chain

**Taproot is currently Bitcoin's "most structured data entry point," but the data is in witness.**

---

## 2.6 Which Fields Are Typically Not Free Data Space?

The following fields are typically not used for embedding arbitrary data:

- **version** - Transaction version number (consensus-critical)
- **locktime** - Time lock (consensus field)
- **value** - Amount (monetary unit)
- **sequence** - Sequence number (see special note below)

### 2.6.1 Special Case: sequence Field

**Design Purpose of sequence Field:**

- Original purpose: Allow transaction replacement before confirmation (deprecated)
- BIP 68: Relative time locks (CheckSequenceVerify)
- Used to control transaction time lock logic

**But There Have Been Historical "Abuse" Cases:**

Early EPOBC (Enhanced Padded Order-Based Coloring) protocol once used sequence's 32-bit space to encode Colored Coins metadata.

**Example:**

```
nSequence = 0xC0L0RFFF

           ↑  ↑
           |  └─ Color ID and amount
           └─ Protocol marker bits
```

**Why Did EPOBC Fail?**

1. ❌ Every transfer requires tracing back to genesis transaction (inefficient)
2. ❌ Cannot verify state without external indexers
3. ❌ Conflicts with BIP 68 time lock functionality
4. ❌ Data space too small (only 32 bits)

**Lesson:** sequence is not designed to store arbitrary data, but it proves an important principle—**people will attempt to exploit every byte in a transaction**.

See Chapter 3 for the complete history of Colored Coins.

---

## 2.7 Consensus vs Policy: Why Are Some Methods Rejected?

Many embedding methods do not violate consensus themselves:

- Witness allows arbitrary bytes
- OP_RETURN size is adjustable at the policy layer
- Multisig pubkeys can technically be any 33 bytes
- Taproot leaves can contain arbitrary script structures

The problem lies at the **policy layer**:

Bitcoin Core can refuse to relay certain transactions, even if they are legal at the consensus layer.

For example:

- Oversized witness
- Non-standard scripts
- Bare multisig
- Meaningless OP_RETURN spam

Therefore, some protocols can "get on-chain" but cannot "enter the public mempool."

This sets the background for understanding protocol controversies like Stamps, Counterparty, and Ordinals in subsequent chapters.

### Specific Boundaries of Consensus vs Policy

**Consensus-allowed but Policy-rejected (Bitcoin Core default) cases:**

1. **Oversized transactions**: Exceeding 400,000 weight units
2. **Non-standard scripts**: Unless wrapped by P2SH/P2WSH/P2TR
3. **Multiple OP_RETURN**: Pre-v30, more than 1 OP_RETURN output per transaction (v30+ allows multiple)
4. **Dust outputs**: Less than 546 satoshis (legacy) or 294 satoshis (SegWit)
5. **Bare multisig**: Bitcoin Core 0.17+ no longer relays

**Purpose of Policy:**

- Prevent UTXO set bloat
- Avoid block space abuse
- Protect node resources

However, miners can ignore these policies and directly package transactions that comply with consensus.

This is why some "non-standard" protocols (like early Stamps) can still get on-chain but cannot propagate through public mempools.

---

## 2.8 Data Entry Capability Comparison Table

| Entry Method | Actual Data Location | Consensus Allows | Policy Allows | Capacity | Weight Cost | Introduction Time | Typical Protocols |
|---------|------------|---------|------------|------|-----------|---------|---------|
| scriptSig | Input scriptSig | ✔ | △ | Medium | 4 WU/byte | 2009 | Colored Coins |
| witness (direct) | Input witness | ✔✔ | ✔✔ | Very large | 1 WU/byte | 2017 (BIP 141) | Ordinals / Atomicals |
| OP_RETURN | **Output** | ✔ | ✔ | 80 bytes (v29) / 100KB (v30+) | 4 WU/byte | 2014 | Omni / Runes |
| multisig fake pubkey | **Output** | ✔ | △ | Medium | 4 WU/byte | 2009 | Stamp |
| P2WSH | **Input witness** | ✔✔ | ✔✔ | Very large | 1 WU/byte | 2017 (BIP 141) | Counterparty (SW) |
| Taproot leaf | **Input witness** | ✔✔ | ✔✔ | Large and structured | 1 WU/byte | 2021 (BIP 341) | Atomicals / RGB |

**Economic Efficiency Ranking:**

1. ⭐⭐⭐⭐⭐ Witness (75% discount) - Including direct witness, P2WSH, Taproot
2. ⭐⭐ OP_RETURN (full base weight)
3. ⭐ Bare multisig (no longer recommended)

**Data Location Classification:**

- **Data in Output**: OP_RETURN, Bare multisig
- **Data in Input Witness**: Direct witness, P2WSH, Taproot

In summary:

- **Witness / P2WSH / Taproot** are the three modern entry points (data in witness, enjoying discount)
- **OP_RETURN** is a direct output entry point (no discount but simple)
- **scriptSig, bare multisig** belong to "historical relics"

---

## 2.9 The Four Eras of Data Embedding

Understanding Bitcoin's data embedding history requires grasping four technical era turning points:

### **Legacy Era (2009-2017): Primitive Exploration**

```
scriptSig → Colored Coins spending path encoding
OP_RETURN → Omni / Counterparty instruction layer
Bare multisig → Early Stamps / Counterparty data containers
```

Characteristics:

- Data mixed with transaction logic
- Prone to UTXO pollution
- Lack of structured design
- Data mainly in output (except scriptSig)

---

### **SegWit Era (2017-2021): Witness Transition Period**

```
witness → Counterparty protocol upgrade
P2WSH → Structured script containers (data in witness)
witness discount → Economic feasibility of large data embedding
```

Characteristics:

- Data decoupled from TXID
- 75% cost discount
- Data migration from output to witness
- Paving the way for next-generation protocols
- This was the transition and exploration period for witness technology

---

### **Taproot Era (2021-2025): Flourishing Diversity**
```
witness + Taproot script-path + ordinal theory → Ordinals inscriptions (data in witness)
BRC-20 → JSON token standard based on Ordinals
witness + Taproot script-path + colored coins mechanism → Atomicals (ARC-20)
Taproot commitments (tapret/opret) → RGB on-chain commitment + off-chain state
Runes → OP_RETURN + UTXO model token protocol (not using Taproot)
```

Characteristics:

- Merkle tree structured data
- Selective reveal (Taproot script-path)
- On-chain commitment, off-chain verification (RGB)
- Witness data enjoys 75% fee discount
- **Protocol flourishing**: Ordinals, BRC-20, Atomicals, RGB, Runes and other protocols emerged successively
- **Technical route differentiation**: Both Taproot + witness innovations (Ordinals/Atomicals) and simplified OP_RETURN solutions (Runes)
- This was the most active period for Bitcoin data embedding

---

### **OP_RETURN v30 Era (2025-now): Policy Turning Point and Discussion**
```
OP_RETURN v30 → 100KB limit + multiple outputs (October 2025)
Bitcoin Core vs Bitcoin Knots → Implementation diversity
Community division → Philosophical debate about Bitcoin's nature
```

Characteristics:

- OP_RETURN expanded from 80 bytes to 100KB
- First time allowing multiple OP_RETURN outputs
- Core developers showing divergent opinions
- Intense community discussion with opposing views
- Addressing divergence through implementation diversity (Bitcoin Core vs Bitcoin Knots) rather than forced uniformity

**Note:** The release of v30 marks an important turning point in Bitcoin's data embedding policy history. This change not only altered technical limitations but also sparked a philosophical debate about Bitcoin's nature—should Bitcoin be pure "electronic cash," or can it carry more data and applications? Discussion continues, and future developments remain to be observed.

Each era represents an upgraded answer to "how to stuff data into Bitcoin."

---

## 2.10 Summary: All Protocols Are Just "Different Combinations of Entry Points"

At this point in Chapter 2, we are ready to read the entire field.

**Core Insight:**

All non-transaction data in Bitcoin originates from input or output.
But more importantly, we need to understand **where data is actually stored**:

- **Data in Output**: Permanently visible upon creation, no cost discount
- **Data in Input Witness**: Revealed when unlocking, enjoying 75% discount

You will see:

- **Omni** → OP_RETURN (data in output)
- **Counterparty** → OP_RETURN + multisig (data in output)
- **Stamp** → multisig (data in output)
- **Ordinals** → Witness (data in witness)
- **Atomicals** → Witness + Taproot Merkle (data in witness)
- **Runes** → Modernized OP_RETURN (data in output)
- **RGB** → Taproot commitment + off-chain state (commitment in output, data off-chain)

Each is:

**Different combinations of input vs output** + **choice of actual data location**.

Understanding this chapter, you can see through these protocols at a glance.

All subsequent chapters of this book will unfold in the following order:

- **Output Data Genealogy:**
  Colored Coins → Omni → Counterparty → Runes
- **Input / Witness Data Genealogy:**
  Stamps bare multisig → P2WSH → Ordinals → Atomicals
- **Taproot Merkle Structured Route**
- **And finally RGB (off-chain state machine)**

These are not isolated events but **the unified history of Bitcoin data embedding**.

---

**After understanding this chapter, you will find when reading subsequent chapters:**

- **Ch3-6 (Output Genealogy)**: How Colored Coins, Omni, Counterparty, Stamps abuse/correctly use scriptPubKey and OP_RETURN (data in output)
- **Ch7-11 (Witness Genealogy)**: How P2WSH, Ordinals, Atomicals push witness to the limit (data in witness)
- **Ch12-13 (Future Directions)**: How RGB completely breaks out of the "data on-chain" mindset

Each chapter will return to the "data entry map" established in this chapter.

The essence of every protocol is **different combinations of these entry points** + **different interpretation layers** + **choice of data location**.

If you understand all possibilities of input and output, and where data is actually stored, you understand the entire history of Bitcoin non-transaction data.

---

**Next Chapter Preview:**

Starting from the earliest Colored Coins, we will see how people gradually discovered and exploited these data entry points, eventually evolving into today's Ordinals, Stamps, and other protocols.

**Experimental Section:** See Chapter 2 Part 2, where we will demonstrate the complete evolution from bare scripts to Commit-Reveal patterns through testnet practice.

