# Chapter 6 — Stamp / Stamps — Bare Multisig as "Image Container"

## Stamp / Stamps — Bare Multisig as "Image Container"

### An Extreme Experiment in Script Structure Abuse

Stamps (broadly referring to the Stamp protocol and related practices) represents an extreme evolution of the Counterparty approach:

Since we can stuff data into multisig public keys, why not stuff the entire data payload?
Instead of just storing parameters, we encode images and content directly as "public keys."

This is a completely "anti-script-semantic" behavior:

- Public keys no longer represent verifiable elliptic curve points
- Multisig is no longer for signatures, but for abusing the structure
- Scripts themselves become a "data packaging format"

This chapter will:

- Structurally explain how Stamps works
- Show how to chunk binary data and stuff it into fake public keys
- Explain why this approach is extremely unfriendly from an engineering perspective
- Explain why it still has "historical significance" and "research value"

---

## 6.1 Stamps' Goal: From State Machine to Pure Data Availability Layer

Recalling the previous chapter: Counterparty used bare multisig to store data, but what was its purpose?

**Counterparty's multisig served the state machine.**

- OP_RETURN carries instruction headers (operation type, asset ID)
- Multisig carries extended parameters (when payload exceeds 80 bytes)
- The meaning of data: **driving state transitions in the off-chain state machine**
- Theoretically, these UTXOs could be spent (though rarely in practice)

What did Stamps do?

**It turned Counterparty's "byproduct" into the "main business."**

Counterparty says: I use multisig to store parameters, and incidentally stuff data in.
Stamps says: I want to store data, multisig is just a container, images are the goal.

This is a fundamental shift:

| | Counterparty | Stamps |
|---|-------------|--------|
| Core Goal | On-chain state machine (asset issuance, transfers) | On-chain permanent storage (images, artwork) |
| Data Role | Parameters for state transitions | Storage itself is the purpose |
| Multisig Purpose | Extend OP_RETURN capacity | Sole data entry point |
| Spending Intent | Theoretically possible | Explicitly not intended (Key Burn) |
| Design Philosophy | Financial protocol | Data availability layer |

Stamps' design intent is clear:

**Make data "permanently" exist in the UTXO set, unprunable.**

This is the "zombie UTXO" strategy:

- Create an output that is theoretically spendable but practically impossible to spend
- Nodes must retain it (because they don't know if it will be spent)
- Data is thus "kidnapped" in the UTXO set, forever unremovable

From Counterparty to Stamps, this is a shift from "protocol designers' reluctant choice" to "intentional exploitation of system resources."

---

## 6.2 Engineering Core: How to Encode Data as "Fake Public Keys"

### Basic Encoding Process

1. **Raw Data Preparation** (e.g., a small image)
   - Compress / re-encode (if needed)
   - Convert to binary data

2. **Data Chunking**
   - Split into fixed-length chunks, 31 bytes per chunk
   - The last chunk may be less than 31 bytes (padding required)

3. **Fake Public Key Encoding**
   - Add `0x02` or `0x03` prefix to each chunk → becomes 33 bytes
   - Prefix choice: typically use `0x02` (indicating even y-coordinate)

4. **Construct Multisig Script**
   ```
   OP_1 <fake_pk_1> <fake_pk_2> ... <fake_pk_n> OP_N OP_CHECKMULTISIG
   ```

### Node Perspective vs. Protocol Parser Perspective

**Node Perspective:**

"This is a valid multisig output."

**Protocol Parser Perspective:**

"This is chunk N of the image or content data."

Each multisig output can carry several 31-byte fragments,
and multiple outputs can be concatenated to form a complete payload.

### Protocol Identification: How Does the Parser Know This Is a Stamp?

Stamps is an independent protocol but uses multisig data encoding techniques similar to Counterparty. The identification process:

1. Scan bare multisig outputs in blocks
2. Check multisig script format (1-of-N multisig)
3. Extract all public key fields, check if they match Stamp encoding patterns
4. If they match, decode fake public keys as data chunks
5. Concatenate data chunks, verify if it's a valid image/content format

Stamps did not invent a new on-chain structure, but rather assigned specific semantics to bare multisig data: **directly interpreting fake public key fields as image data containers**.

---

## 6.3 Stamps' Data Recovery Process (Parser Perspective)

The parser's process when scanning blocks:

1. **Discover Multisig Outputs with Specific Format** (Stamp-agreed pattern)
   - Identify bare multisig scripts
   - Check if they conform to Stamp protocol format

2. **Read Each `<pubkey>`**
   - Extract all public key fields
   - Ignore other parts of the script

3. **Data Extraction**
   - Discard the first byte (0x02/0x03), obtain 31-byte raw data chunk
   - Remove padding (if any)

4. **Data Concatenation**
   - Concatenate all data chunks in predetermined order
   - Restore the original data stream

5. **Content Interpretation**
   - Interpret the concatenated byte stream as image / text / other content
   - Verify data integrity

From an engineering perspective, this process is similar to "recovering data from P2WSH scripts,"
but it:

- Completely ignores signatures
- Completely ignores elliptic curve validity
- Completely treats script fields as "portable hard drives"

### Real Transaction Case Analysis

**Transaction:** `b1278acf50c27342753d01af9013a709509fdd920bc40ddcbec0738aaec2764c`

This is a real Stamp transaction. Observe its output addresses:

```
bc1qapfywj2x8qukztqpcgqlxqqqqqqqqqrxqqq9zhp84vqqpq5t8lyqfvkk5y
bc1q5m4ywq8lcgq0hlxylh7axqqqqqqqqqqqqqqqqqqqqqqqqqqqqqsskk5xx9
```

**Note the large number of repeated "q" characters in the addresses**—in bech32 encoding, `q` represents 0. Many consecutive `q`s mean the original data contains many zero bytes or repeating patterns, which is a typical characteristic of encoded image data.

scriptPubKey structure (simplified):

```
51                          # OP_1 (1-of-N)
21                          # PUSH 33 bytes
02[31 bytes of image data]  # fake pubkey 1
21                          # PUSH 33 bytes
02[31 bytes of image data]  # fake pubkey 2
...
53                          # OP_3 (N=3)
ae                          # OP_CHECKMULTISIG
```

Key point: **In scriptPubKey, public keys are merely pushed onto the stack; elliptic curve validity is not verified.** Only when someone attempts to spend will `OP_CHECKMULTISIG` perform signature verification—but since these "public keys" have no corresponding private keys, no one will attempt it, and verification will never occur.

**Key Burn Mechanism**: The Stamps protocol later introduced Key Burn, setting one of the multisig public keys to a known burn address (e.g., `0222...2222`), proving that the UTXO can never be spent, making Stamp's "permanence" more explicit.

---

## 6.4 Comparison with Counterparty: From "Extended Data" to "Pure Data Container"

| Protocol | Multisig Purpose | OP_RETURN Purpose | Data Location |
|----------|------------------|-------------------|---------------|
| Counterparty | Extended parameters, structured payload | Instruction header (type, property, etc.) | Dual channel |
| Stamps | Direct storage of all data | Possibly only for location or metadata (optional) | Primarily in multisig |

It can be seen that Stamps took a step further:

**Completely "de-financialized" script data space, turning it into pure content storage.**

From Bitcoin Core policy perspective, this is a very dangerous trend—
if everyone does this, the UTXO set and block data will rapidly bloat.

---

## 6.5 Engineering Reproduction: Stamp Data Encoding and Decoding

> **💡 Complete Code Implementation:** This section shows core code snippets. The complete runnable implementation is located in the `code/stamps/` directory.

Stamp's data encoding implementation consists of two core parts: data chunking encoding and multisig script construction. The encoding function chunks raw data into 31-byte blocks, adds a `0x02` prefix to each chunk to convert it into a 33-byte fake compressed public key; then groups the fake public keys and constructs a 1-of-N bare multisig script. The decoding function extracts all public key fields from the multisig script, removes prefixes, and concatenates them to restore the original data.

The parser needs to scan bare multisig outputs in blocks, identify scripts conforming to Stamp format, extract fake public key fields and decode them as data chunks, then concatenate them in order to restore the complete image or content. This design allows Stamp to embed arbitrary binary data into the UTXO set's scriptPubKey, but at the cost of permanently occupying UTXO set space. The complete implementation includes error handling, boundary checks, and data integrity verification. See the `code/stamps/` directory for details.

---

## 6.6 Stamps' Problems: Why Is It Considered "Extreme Spam"?

### 6.6.1 From Core's Perspective

- These multisig UTXOs are theoretically spendable
- But in practice, no one will spend them (private keys don't exist)
- Causes UTXO set pollution (permanent "zombie UTXOs")
- Node storage and sync burden forced to carry garbage

**Policy Layer Response:**

- **Bitcoin Core 0.13.0 (August 2016)**: Introduced `-permitbaremultisig` option, allowing users to choose to reject relaying bare multisig transactions (default still allows)
- **Bitcoin Core 0.17.0 (October 2018)**: Bare multisig outputs are no longer automatically considered "IsMine" by wallets, even if all private keys are in the wallet

### 6.6.2 From Protocol Engineering Perspective

- This usage has no script semantic value
- Difficult to distinguish from standard multisig transactions (requires parser heuristics)
- Hard to maintain "data/non-data" distinction through policy
- Makes script fields "semantically degenerate" into pure byte arrays
- Parsers must scan all multisig outputs to determine if they are Stamps

### 6.6.3 Resource Consumption Problem

```python
# Example: Storing a 100KB image

image_size = 100 * 1024  # 100 KB
bytes_per_chunk = 31
chunks_needed = (image_size + bytes_per_chunk - 1) // bytes_per_chunk
outputs_needed = (chunks_needed + 19) // 20  # Assuming 20 keys per output

print(f"Requires {outputs_needed} multisig outputs")
print(f"Each output occupies UTXO set space")
print(f"Total: {outputs_needed} permanent UTXOs")

# If BTC = $40,000, each output requires at least 600 satoshis
cost = outputs_needed * 600 / 100000000 * 40000
print(f"Cost: ${cost:.2f}")
```

Therefore:

**Stamps became one of the focal points of debate between mempool policy and "should Bitcoin be an image hosting service."**

### 6.6.4 OLGA: Stamps' Evolution

In February 2024 (block height 833,000), the Stamps protocol introduced the **OLGA** (Optimized Lightweight Graphical Asset) format:

| Feature | Classic Stamp | OLGA |
|---------|---------------|------|
| Script Type | Bare multisig (P2MS) | P2WSH |
| Encoding | Base64 | Raw binary |
| Transaction Size | 100% | ~50% |
| Cost | 100% | ~30-40% |
| Max Capacity | ~7-8 KB | ~65 KB |

OLGA's emergence marks Stamps protocol's "self-correction": acknowledging the cost advantages of the witness route while maintaining the core philosophy.

---

## 6.7 Stamps' Significance: A "Boundary Marker" at the Extreme Point

Although from an engineering and resource utilization perspective, it is an unwelcome practice,
from the perspective of "non-transaction data history," it marks a critical boundary:

**If clear boundaries are not established, any script field will eventually become a data warehouse.**

### Stamps' Historical Significance

1. **Marked the extreme point of the multisig route**
   - Pushed Counterparty's approach to the extreme
   - Proved the feasibility of "script as storage" (though not recommended)

2. **Forced the community to directly discuss "what is resource abuse"**
   - Triggered discussions about UTXO set management
   - Promoted policy layer improvements

3. **Paved the way for witness / Taproot to be accepted as more reasonable data entry points**
   - Demonstrated the limitations of script fields as data containers
   - Proved the need for better data embedding mechanisms

### Subsequent Evolution

Stamps is the extreme point of the multisig route,
and will gradually be replaced by:

- **P2WSH / witness embedding** (data in witness, enjoying 75% discount)
- **Taproot Merkle route** (structured, verifiable)
- **Client-side validation (RGB)** (off-chain state + on-chain commitment)

---

## 6.8 Stamp's Position in This Book: The Extreme Point of Script Abuse

If we say:

- **Colored Coins** is a zero-byte protocol
- **Omni** is a single-channel explicit data protocol
- **Counterparty** is a dual-channel embedding protocol

Then **Stamp** is:

**The extreme point of script structure abuse.**

### Position in Data Embedding History

| Protocol | Data Entry | Characteristics | Problems |
|----------|------------|-----------------|----------|
| Counterparty | OP_RETURN + bare multisig | Dual channel, structured | Occupies UTXO |
| **Stamp** | **Bare multisig** | **Single channel, extreme** | **Severe UTXO pollution** |
| P2WSH | witness | Data in witness | More reasonable |
| Ordinals | witness | Large capacity | Enjoys discount |

### Its Importance Lies In

- Forcing the community to directly discuss "what is resource abuse"
- Bringing the "script fields as data containers" route to its end
- Paving the way for witness / Taproot to be accepted as more reasonable data entry points
- Proving the need for better data embedding mechanisms

---

## 6.9 Summary: Stamp's Engineering Significance

Stamp represents:

- ✔ Extreme practice of script structure "semantic abuse"
- ✔ The sign that the multisig route has reached its end
- ✔ Exposed the fundamental problems of script fields as data containers
- ✔ Paved the way for subsequent witness / Taproot routes
- ✔ Defined the boundary of "resource abuse" in controversy

And points the direction for future development:

**Script fields are not suitable as data containers.**
**Better data embedding mechanisms are needed.**

This leads to the next chapter:

**Chapter 7 — P2WSH — Script Becomes a "Legitimate Data Container"**

From abusing scripts to structured containers, P2WSH places data in witness, enjoying a 75% cost discount, becoming a more reasonable data embedding approach.

---

