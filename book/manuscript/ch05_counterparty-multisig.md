# Chapter 5 — Counterparty — The First Systematic Practice of Bare Multisig as Data Container

## Counterparty — The First Systematic Practice of Bare Multisig as Data Container

### Why Counterparty?

After Omni, multiple protocols emerged on Bitcoin attempting to expand data capacity. Among them, Counterparty (launched in January 2014) was **the first protocol to systematically and extensively use bare multisig as a data container**.

Although there may have been sporadic experiments before it, Counterparty standardized, engineered, and applied this technique at scale in practice. Its multisig data encoding scheme directly inspired the Stamp protocol, becoming the pioneering work of the "bare multisig as storage" technical approach.

More importantly, it had real user adoption and on-chain data during 2014-2016:
- Issued over 50,000+ assets
- Supported early NFT projects like Rare Pepes (2016)
- Operated a decentralized exchange (DEX)
- Was the most complete "on-chain asset protocol" before Ethereum

This chapter focuses on Counterparty's core technical contribution: **how to encode data into multisig public key fields**, and the advantages and costs of this technique.

---

## 5.1 Design Goals: Not Just Assets, but a "Protocol Stack"

### From "Explicit Data" to "Structured Payloads"

After Omni, people had accepted a fact:

**To implement assets and protocols on Bitcoin, explicit data must be written.**

But Omni primarily relied on a single channel: OP_RETURN.
Its payload was limited (80 bytes), structurally simple, and insufficient for expressing complex state machines.

Counterparty went further—it made two bold engineering attempts:

1. Continue using OP_RETURN for protocol instructions
2. **Leverage multisig public key fields (bare multisig) to embed more structured data**

It thus became:

- **The pioneer and standard-setter for bare multisig data containers**
- The direct ancestor of the Stamp protocol
- The inspiration source for subsequent Ordinals/Atomicals "embedding data into unintended fields"
- The first mature example of multi-channel data embedding

### Counterparty's Ambition

Counterparty's ambition extended beyond "issuing a token" to:

- Issuing assets (Tokens)
- Implementing decentralized exchange (DEX)
- Supporting programmable instructions (bets, contracts, etc.)
- Minimizing changes to Bitcoin consensus

In this book's terms:

Omni moved the "state machine" off-chain,
while Counterparty attempted to carry richer "instruction languages" on-chain.

For this, it needed two things:

1. A clean instruction channel (OP_RETURN)
2. A higher-capacity payload channel (multisig public key fields)

---

## 5.2 OP_RETURN Channel: Instruction Header + Protocol Identification

Similar to Omni, Counterparty uses:

```
OP_RETURN <payload>
```

The payload contains:

- Protocol identification (magic: `CNTRPRTY`)
- Instruction type (type: create, transfer, order placement, etc.)
- Asset ID or issuance information
- Amount/quantity fields
- Some additional parameters

This part primarily serves:

**"Here is a Counterparty protocol message. What type is it?"**

But Counterparty quickly encountered a problem:

**OP_RETURN's byte count is insufficient.**

To carry more complex data structures, it needed to find a second outlet.

---

## 5.3 Bare Multisig Channel: Encoding Data as "Fake Public Keys"

### Multisig Script Structure

Multisig script format:

```
OP_1 <pubkey1> <pubkey2> ... <pubkeyN> OP_N OP_CHECKMULTISIG
```

Each `<pubkey>` pushed onto the stack only needs to satisfy at the consensus layer:

- A 33-byte (compressed public key) or 65-byte (uncompressed public key) string
- Elliptic curve point validity verification is only checked to a limited extent

Thus, Counterparty made a crucial engineering decision:

**Encode arbitrary data as "fake public keys" and embed them into multisig script public key positions.**

### Fake Public Key Encoding Principle

**Compressed public key format (33 bytes):**

```
0x02/0x03 + 32 bytes data
```

- `0x02` or `0x03` is the compressed public key prefix (indicating y-coordinate parity)
- The following 32 bytes can be arbitrary data

**Uncompressed public key format (65 bytes):**

```
0x04 + 64 bytes data
```

- `0x04` indicates uncompressed public key
- The following 64 bytes can be arbitrary data

### Why Fake Public Keys Work: Consensus Validation Behavior

A critical detail that makes this technique possible:

**Bitcoin consensus does not check whether a pushed "public key" is a valid elliptic curve point unless it participates in a signature check.**

When a public key is pushed onto the stack in a script, Bitcoin's consensus rules only verify:
- The byte length (33 bytes for compressed, 65 bytes for uncompressed)
- The format prefix (0x02/0x03/0x04)

Full elliptic curve point validation only occurs during signature verification operations (OP_CHECKSIG, OP_CHECKMULTISIG, etc.), when the public key is actually used to verify a signature.

Counterparty exploits this behavior: since the fake public keys in the multisig output are never used for signature verification (only the real public key is), they can be arbitrary 32-byte payloads with a valid prefix. The script remains consensus-valid, and the data remains embedded.

### Counterparty's Typical Implementation

Counterparty uses **1-of-3 multisig**:

```
OP_1 <fake_pubkey1> <fake_pubkey2> <real_pubkey> OP_3 OP_CHECKMULTISIG
```

- **First 2 public keys**: Fake public keys encoding data
- **3rd public key**: Real public key for reclaiming BTC

From Bitcoin Core's perspective, this output is simply:

**A valid multisig output**

But from Counterparty's parser perspective:

**This is a data carrier storing protocol payloads**

This extends the protocol's expressiveness:

- Larger capacity (33 bytes per public key, 2 public keys = 64 bytes)
- Can be split across multiple public key positions
- Supports complex parameters and structured payloads

---

## 5.4 Dual-Channel Collaboration: Instruction Header + Payload Body

Typical pattern:

- **OP_RETURN** handles "what instruction is this" (instruction type, basic parameters)
- **Multisig public key fields** handle "complete parameters/data for this instruction" (extended data)

Metaphorically:

- OP_RETURN is the HTTP header
- Multisig payload is the HTTP body

### Parsing Flow

When parsing, Counterparty clients will:

1. Scan OP_RETURN in the transaction to confirm this is a Counterparty message
2. Parse multisig outputs in the same transaction, decoding public key fields as data
3. Combine into a complete protocol instruction
4. Update the off-chain state machine (who owns how many assets, who placed what orders)

### Example: Asset Issuance Transaction Structure

```
Transaction structure:
  output[0]: OP_RETURN <counterparty-header>
    - magic: "CNTRPRTY"
    - type: "issuance"
    - asset_name: "MYTOKEN"
  
  output[1]: OP_1 <fake_pubkey1> <fake_pubkey2> <real_pubkey> OP_3 OP_CHECKMULTISIG
    - fake_pubkey1: 0x02 + 32 bytes (asset description data)
    - fake_pubkey2: 0x02 + 32 bytes (metadata extension)
    - real_pubkey: Real public key for reclaiming dust
```

The parser combines these to obtain the complete asset issuance instruction.

### Channel Comparison: OP_RETURN vs Bare Multisig

Counterparty's dual-channel approach leverages two fundamentally different data embedding mechanisms. The following table compares their characteristics:

| Channel | Capacity | Stored in UTXO? | Spendability | Policy Risk | Attack Surface |
|---------|----------|-----------------|--------------|-------------|----------------|
| OP_RETURN | 80 bytes | No | None | Low | Minimal |
| Bare Multisig | 33×N bytes | Yes | Hard | High | Malformed scripts, UTXO bloat |

**Key Differences:**

- **OP_RETURN**: Provably unspendable, does not pollute UTXO set, but limited to 80 bytes. Bitcoin Core policy explicitly supports it as a data channel.
- **Bare Multisig**: Higher capacity (33 bytes per public key, scalable), but permanently occupies UTXO set, difficult to spend, and faces increasing policy restrictions.

Counterparty's innovation was combining both: using OP_RETURN for protocol identification and basic parameters, while leveraging bare multisig for extended payloads. This hybrid approach maximized capacity while maintaining some level of policy acceptance.

---

## 5.5 Core Encoding Principle Summary

Counterparty's bare multisig data encoding can be summarized in three core steps:

### Step 1: Data Chunking

Chunk the data to be encoded into 32-byte blocks:

```
Original data: [200 bytes]
After chunking:
  - chunk1: 32 bytes
  - chunk2: 32 bytes
  - chunk3: 32 bytes
  - ...
```

### Step 2: Encode as Fake Public Keys

Add public key prefix to each data chunk:

```
chunk1 → 0x02 + chunk1 = fake_pubkey1 (33 bytes)
chunk2 → 0x02 + chunk2 = fake_pubkey2 (33 bytes)
...
```

### Step 3: Construct Multisig Script

Combine fake public keys with real public key into multisig script:

```
OP_1 
  <fake_pubkey1> 
  <fake_pubkey2> 
  <real_pubkey>
OP_3 
OP_CHECKMULTISIG
```

Thus, Counterparty achieves:

- **Consensus layer**: This is a valid 1-of-3 multisig output
- **Protocol layer**: This is a container holding 64 bytes of data

---

## 5.6 Engineering Reproduction: Dual-Channel Data Embedding

> **💡 Complete Code Implementation:** This section shows core code snippets. The complete runnable implementation is located in the `code/counterparty/` directory.

Counterparty's dual-channel data embedding implementation consists of two core parts: fake public key encoding/decoding and transaction construction/parsing. The encoding function adds a `0x02` prefix to arbitrary 32-byte data to convert it into a 33-byte fake compressed public key; the decoding function extracts the original data in reverse. During transaction construction, the OP_RETURN output carries the protocol instruction header, while the multisig output carries the extended data payload, combining to form a complete protocol message.

The parser must simultaneously scan OP_RETURN and multisig outputs, decode the public key fields, and concatenate them with OP_RETURN data to restore the complete protocol instruction. This dual-channel design enables Counterparty to break through OP_RETURN's 80-byte limit, supporting more complex scenarios like asset issuance and DEX orders. The complete implementation includes error handling, boundary checks, and type validation. See the `code/counterparty/` directory for details.

---

## 5.7 Counterparty's Rise and Fall: From Glory to Marginalization

### 5.7.1 Peak Period (2014-2016)

Counterparty achieved significant success in its early days:

**Technical Achievements:**
- Launched in January 2014, distributed XCP through Proof-of-Burn destroying 2,140 BTC
- First to implement complete DEX + asset issuance + smart contracts on Bitcoin
- July 2014: Overstock.com's tZERO project chose Counterparty
- 2015: Spells of Genesis (first on-chain game assets)
- 2016: Rare Pepes (pioneer of NFT concept, one year before CryptoPunks)

**On-Chain Data:**
- Cumulative assets issued: 50,000+
- Peak daily transaction volume: thousands of transactions
- Most active meta-protocol on Bitcoin during 2014-2016

### 5.7.2 Reasons for Decline (2017+)

**1. Ethereum's Rise (2017-2018)**
- ERC-20 standard was simpler and more flexible
- Gas model was more predictable than Bitcoin transaction fees
- Smart contract functionality was more powerful
- Developer ecosystem rapidly shifted to Ethereum

**2. Bitcoin Policy Restrictions (2016+)**
- Bitcoin Core 0.13.0 (2016) introduced the `-permitbaremultisig` option
- Nodes could choose to reject bare multisig transactions
- Modern policy only accepts multisig up to 3-of-N
- Directly limited Counterparty's multisig data containers

**3. High Bitcoin Transaction Fees (2017 Bull Market)**
- Bitcoin fees surged to $50+/tx by late 2017
- Counterparty transactions required multiple outputs, increasing costs
- Users migrated to cheaper chains

**4. Lack of Killer App**
- Omni had USDT (true product-market fit)
- Counterparty's art/collectibles market was too niche
- Rare Pepes had historical significance but limited commercial value

### 5.7.3 Current Status

- Protocol still operates but activity has significantly declined
- Migrated to modern encoding methods like P2TR (bare multisig deprecated)
- Main value: historical significance + NFT collectibles market

---

## 5.8 Advantages and Costs: Structural Issues Exposed by Counterparty

### Advantages

- Leverages existing structure (multisig public key fields) to expand capacity
- Requires no new opcodes
- Does not change consensus
- Protocol is iterable
- Expressiveness significantly stronger than pure OP_RETURN

### Costs

#### 1. Bare Multisig = Policy Layer's Unwelcome Pattern

- Occupies UTXO set
- Difficult to spend (theoretically requires multiple signatures, though Counterparty uses 1-of-N to circumvent)
- Confuses script semantics (data masquerading as keys)
- Bitcoin Core 0.13.0+ (2016) introduced restrictions; nodes can choose to reject bare multisig
- Modern policy only accepts multisig up to 3-of-N

#### 2. Block/UTXO Resources Used as Data Storage

- Places significant burden on nodes
- Conflicts with the "UTXO should be minimal" philosophy
- Permanently occupies UTXO set space (though reclaimable in theory, many remain unreclaimed in practice)

#### 3. Increased Parser Complexity

- Must simultaneously parse OP_RETURN and multisig
- Must defend against malformed data, attack payloads, malicious scripts
- Needs to handle various multisig script variants

#### 4. Established Precedent for "Abusing Scripts as Storage"

- Stamp and some Ordinals-style attempts directly inherited this approach
- Opened the "script as storage" design paradigm

---

## 5.9 Counterparty's Position in This Book: First Abuse of Multi-Channel Embedding and Script Structure

If we say:

- **Colored Coins** is a zero-byte protocol
- **Omni** is a single-channel explicit data protocol

Then **Counterparty** is:

**A dual-channel embedding + structured payload protocol.**

### Its significance has three layers:

1. **Proved that OP_RETURN is insufficient; protocols actively seek second outlets**
   - When single-channel capacity is insufficient, protocol designers seek other data entry points
   - This laid the foundation for subsequent use of witness and Taproot

2. **Opened the Pandora's box of "multisig fields as data containers"**
   - First systematic use of script fields as data storage
   - Proved the feasibility of "consensus-allowed but semantically abusive" approaches

3. **Established practical foundation for subsequent Stamps, bare multisig, and script-hidden data**
   - Stamp protocol directly inherited Counterparty's multisig data container approach
   - Ordinals and Atomicals were also inspired by this "unintended field utilization"

### Position in Data Embedding History

| Protocol | Data Entry | Characteristics |
|----------|------------|-----------------|
| Colored Coins | None (zero bytes) | Pure interpretation layer |
| Omni | OP_RETURN | Single channel, 80 bytes |
| **Counterparty** | **OP_RETURN + bare multisig** | **Dual channel, structured** |
| Stamp | Bare multisig | Single channel, extreme |
| Ordinals | Witness | Single channel, large capacity |
| Atomicals | Witness + Taproot | Multi-channel, structured |

---

## 5.10 Summary: Counterparty's Engineering Significance

Counterparty represents:

- ✔ The first mature example of multi-channel data embedding
- ✔ The first systematic practice of "semantic abuse" of script structures
- ✔ Proof that protocol designers actively seek data entry points
- ✔ Technical path provided for subsequent protocols (Stamp, Ordinals)
- ✔ Exposed advantages and disadvantages of bare multisig as data containers

And points the direction for future development:

**When single channels are insufficient, protocols seek second and third channels.**
**But each channel has its costs and limitations.**

This leads to the next chapter:

**Chapter 6 — Stamp — The Extremization of Bare Multisig**

Stamp pushed Counterparty's multisig data container approach to the extreme, relying entirely on bare multisig, becoming the typical representative of "multisig as storage."

---

