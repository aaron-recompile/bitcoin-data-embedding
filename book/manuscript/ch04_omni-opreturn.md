# Chapter 4

## Omni (Mastercoin) — The First Formal Application of OP_RETURN

### The "First-Generation Explicit Protocol" for Bitcoin On-Chain Data Embedding

During the Colored Coins era, people attempted to build assets on-chain but dared not write any data.
It was fragile in engineering and easily broken by wallet logic.
The failure of Colored Coins made everyone realize:

**Without writing "explicit data" on-chain, a true state machine cannot exist.**

Thus, the next generation of protocols took the stage:
**Mastercoin (later renamed Omni)**.

This was the first protocol in Bitcoin's history to explicitly, systematically, and standardly rely on on-chain data.
It pioneered:

- The concept of on-chain instructions (scriptless instructions)
- The architectural pattern of off-chain state machine + minimal on-chain commitment
- The foundation for USDT to run on Bitcoin
- Pushing OP_RETURN to become the standard channel for Bitcoin data embedding

This chapter will reconstruct Omni from both historical and engineering perspectives, explaining how it became the first asset protocol to "explicitly write data."

---

## 4.0 Review: Omni's Choice from the "Data Entry Map" Perspective

In Chapter 2, we established a complete map of Bitcoin data embedding:

**Omni's Choice:**

- ✅ Uses OP_RETURN on the Output side (data in scriptPubKey)
- ✅ Data is permanently visible upon creation
- ✅ Does not occupy UTXO set (provably unspendable)
- ❌ Cannot enjoy witness's 75% discount (OP_RETURN in 2014, witness in 2017)
- ❌ Capacity limited (40-80 bytes)

**Why did Omni choose this path?**

Historical context (2013-2014):

- SegWit had not yet appeared (activated in 2017)
- Witness discount did not exist
- OP_RETURN was the only "recognized" data entry point
- 80 bytes was sufficient for simple instructions

**Omni's position in the evolution of data embedding:**

```
Colored Coins (2012)          → No data, pure interpretation
  ↓
Mastercoin (2013)            → Multisig fake pubkeys (UTXO pollution)
  ↓
OP_RETURN (2014)             → Bitcoin Core response
  ↓
Omni (2014-now)              → OP_RETURN standardization
  ↓
Counterparty (2014)          → OP_RETURN + multisig hybrid
  ↓
SegWit (2017)                → Witness emergence
  ↓
Ordinals (2023)              → Witness + Taproot
```

---

## 4.1 Mastercoin's Ambition: Building a "Second-Layer Financial System" on Bitcoin

In 2013, the Mastercoin whitepaper proposed a bold vision:

"Build new assets, smart contracts, stablecoins, and decentralized exchanges on Bitcoin."

Its goals far exceeded Colored Coins, including:

- Custom token issuance
- Decentralized exchange (DEX)
- Crowdsale
- Multi-asset layers
- Contract instructions

Ethereum did not exist at the time, nor did Solidity.
**Mastercoin was the world's first on-chain meta-protocol.**

---

## 4.2 Mastercoin's Data Embedding Evolution

**Important historical fact: Mastercoin did not initially use OP_RETURN!**

Mastercoin's data embedding method went through three stages:

### Stage 1: Multisig Public Key Embedding (2013-early 2014)

**Historical Background:**

- Mastercoin whitepaper published in **January 2012**
- ICO in **July 2013**
- OP_RETURN was not yet formally supported by Bitcoin Core

**Technical Implementation:**

- Initially used 1-of-3 multisig, encoding data as fake public keys
- Each "public key" was actually encoded protocol data
- This led to UTXO set pollution issues

**Example:**

```
OP_1 <fake_pubkey1> <fake_pubkey2> <fake_pubkey3> OP_3 OP_CHECKMULTISIG
```

Where `<fake_pubkey>` is actually Mastercoin protocol data.

**Problems:**

- ❌ Occupies UTXO set (permanent pollution)
- ❌ Difficult to spend (requires multiple signatures)
- ❌ Confuses script semantics (data disguised as keys)
- ❌ Raised concerns among Bitcoin Core developers

### Stage 2: Transition Period (2014)

**Bitcoin Core's Response:**

- **Bitcoin Core v0.9.0 (March 2014)** introduced OP_RETURN
- Initial limit: **40 bytes**
- Purpose: **Respond to Mastercoin and Counterparty's multisig data embedding methods**

**Bitcoin Core Developers' Considerations:**

- Rather than letting protocols pollute the UTXO set (through unspendable multisig outputs)
- Provide a formal data embedding channel (OP_RETURN)

**Mastercoin's Migration:**

- Began gradual migration from multisig to OP_RETURN
- But the 40-byte limit was still tight

### Stage 3: OP_RETURN Standardization (Post-2014)

- **v0.11.0 (April 2015)** increased to **80 bytes**
- Mastercoin renamed to **Omni Layer**
- Complete migration to OP_RETURN + reference output pattern

**The Importance of This Historical Detail:**

1. It explains why Counterparty also used multisig (influenced by Mastercoin's early implementation)
2. It shows that OP_RETURN's introduction was partly in response to Mastercoin's UTXO pollution problem
3. This connects the technical evolution from Chapter 3 (Colored Coins) to Chapter 5 (Counterparty)

**Key Understanding:**

- Mastercoin **promoted** the adoption of OP_RETURN
- But OP_RETURN was **not** Mastercoin's original invention
- Rather, it was **Bitcoin Core's response to meta-protocol needs**

---

## 4.3 Omni's Architecture: On-Chain Log + Off-Chain State Machine

Omni adopted a very innovative design at the time (2013):

### Architecture Diagram

```
┌─────────────────────────────────────────────┐
│           Bitcoin Blockchain                 │
│  (Stores only OP_RETURN instructions, no state) │
│                                              │
│  Block N:   [OP_RETURN: Transfer 10 USDT]  │
│  Block N+1: [OP_RETURN: Transfer 5 USDT]   │
│  Block N+2: [OP_RETURN: Issue Token X]     │
└─────────────────────────────────────────────┘
                      ↓
                      ↓ Scan and parse
                      ↓
┌─────────────────────────────────────────────┐
│          Omni State Machine                  │
│   (Maintains complete state of all assets)   │
│                                              │
│  Address A:  100 USDT                       │
│  Address B:  50 USDT                        │
│  Token X:    Total Supply 1000              │
└─────────────────────────────────────────────┘
```

### Key Principles

**1. On-Chain is Only "Event Log" (Event Log)**

- Each OP_RETURN is a log entry
- Records "what happened," not "what the current state is"
- Similar to a database's Write-Ahead Log

**2. Off-Chain Maintains "Current State" (Current State)**

- Omni nodes scan from the genesis block
- Execute each instruction in order
- Accumulate to calculate current state

**3. State Consistency Comes from Instruction Order**

- Bitcoin blockchain guarantees global order of instructions
- All honest Omni nodes execute the same instruction sequence
- Arrive at the same state result

### Why This Design?

**Advantages:**

- Minimal on-chain data (only instructions, no state)
- High flexibility (can modify state machine logic)
- Simple on-chain verification (only need to verify transaction validity)

**Disadvantages:**

- Must scan complete history to derive current state
- Difficult to implement light clients
- State machine bugs can cause consensus splits
- Cannot verify Omni state at Bitcoin consensus layer

**This design profoundly influenced subsequent protocols:**

- Counterparty: Same pattern
- Runes: Same pattern
- BRC-20: Same pattern
- RGB: Improved version (client-side validation)

### Instruction Type Examples

Omni clients parse instruction types in OP_RETURN:

- Instruction 0x00 → Simple Send (transfer)
- Instruction 0x32 → Create Property (issue asset)
- Instruction 0x33 → Grant Tokens (grant tokens)
- Instruction 0x34 → Revoke Tokens (revoke tokens)
- Instruction 0x50 → DEX Sell Offer (DEX listing)
- Instruction 0x51 → DEX Accept Offer (DEX accept order)

---

## 4.4 Omni's Data Format (Engineering Analysis)

Omni's data fields are encoded as:

```
OP_RETURN <payload>
```

Payload structure:

| Field | Length | Description |
|------|------|------|
| magic | 4 bytes | Protocol identifier ("omni" = 0x6f6d6e69) |
| version | 2 bytes | Protocol version number |
| type | 2 bytes | Instruction type (e.g., issuance, transfer, DEX, etc.) |
| property_id | 4 bytes | Asset ID |
| amount | 8 bytes | Amount |
| extra fields | Variable | Varies by type |

### Example 1: Simple Send (Transfer)

```
OP_RETURN 6f6d6e69000000000000001f000000003b9aca00
```

Parsing:

```
6f6d6e69          = "omni" (magic bytes)
0000              = version (0)
0000              = type (0 = Simple Send)
0000001f          = property_id (31 = USDT)
000000003b9aca00  = amount (1,000,000,000 = 10.00000000 USDT)
```

### Example 2: Create Property (Issue Asset)

```
OP_RETURN 6f6d6e690000003200000000...
```

Parsing:

```
6f6d6e69          = "omni" (magic bytes)
0000              = version (0)
0032              = type (0x32 = Create Property)
00000000          = property_id (0 = new asset)
...               = Asset name, description, and other metadata
```

**Omni Protocol Version Evolution:**

- **Omni Layer v0**: Original Mastercoin protocol
- **Omni Layer v1**: Simplified protocol, mainly for USDT
- Currently primarily uses v1, focused on stablecoin functionality

---

## 4.5 Complete Engineering Reproduction: Building Omni Transactions

> **💡 Complete Code Implementation:** This section shows core code snippets. Complete runnable implementation is located in the `code/omni/` directory.

### Prerequisites: Omni's Two-Layer Structure

Omni transactions exist simultaneously at two levels:

1. **Bitcoin Layer**: Normal Bitcoin transaction (transfers Bitcoin)
2. **Omni Layer**: Protocol-layer asset transfer (transfers Omni assets)

### Example: Sending 10 USDT

**Step 1: Build Bitcoin Transaction**

```python
# Using bitcoinlib or similar tools
from bitcoinlib.transactions import Transaction

tx = Transaction()

# Add input
tx.add_input(sender_utxo)

# Add OP_RETURN output (Omni payload)
omni_payload = build_omni_simple_send(31, 1000000000)  # 10 USDT
tx.add_output(
    script_pubkey=create_op_return_script(omni_payload),
    value=0  # OP_RETURN output value is 0
)

# Add receiver output (dust)
tx.add_output(
    script_pubkey=receiver_address,
    value=546  # dust limit
)

# Add change output
tx.add_output(
    script_pubkey=change_address,
    value=utxo_value - 546 - fee
)
```

**Step 2: Build Omni Payload**

```python
import struct

def build_omni_simple_send(property_id, amount):
    """
    Build Omni Simple Send instruction
    
    Args:
        property_id: Asset ID (31 = USDT)
        amount: Amount in smallest units (1 USDT = 100,000,000)
    
    Returns:
        payload bytes
    """
    payload = bytearray()
    
    # Magic bytes: "omni"
    payload.extend(b'omni')  # 0x6f6d6e69
    
    # Transaction version (0)
    payload.extend(struct.pack('>H', 0))
    
    # Transaction type (Simple Send = 0)
    payload.extend(struct.pack('>H', 0))
    
    # Property ID (31 for USDT)
    payload.extend(struct.pack('>I', property_id))
    
    # Amount (8 bytes, divisible tokens)
    payload.extend(struct.pack('>Q', amount))
    
    return bytes(payload)

def create_op_return_script(payload):
    """
    Create OP_RETURN script
    
    Args:
        payload: Omni payload bytes
    
    Returns:
        scriptPubKey bytes
    """
    script = bytearray()
    script.append(0x6a)  # OP_RETURN
    script.append(len(payload))  # Length
    script.extend(payload)
    return bytes(script)

# Send 10.00000000 USDT
# 1 USDT = 100,000,000 smallest units
# 10 USDT = 1,000,000,000 smallest units
payload = build_omni_simple_send(31, 1000000000)
# Result: 6f6d6e69 0000 0000 0000001f 000000003b9aca00
```

**Step 3: Parse Omni Transaction**

```python
def parse_omni_transaction(tx):
    """
    Parse Omni transaction
    
    Args:
        tx: Bitcoin transaction object
    
    Returns:
        Omni instruction dictionary
    """
    # Find OP_RETURN output
    for output in tx.outputs:
        script = output.script_pubkey
        if script.startswith(b'\x6a'):  # OP_RETURN
            length = script[1]
            payload = script[2:2+length]
            
            # Check magic bytes
            if payload[:4] != b'omni':
                continue
            
            # Parse payload
            version = struct.unpack('>H', payload[4:6])[0]
            tx_type = struct.unpack('>H', payload[6:8])[0]
            property_id = struct.unpack('>I', payload[8:12])[0]
            amount = struct.unpack('>Q', payload[12:20])[0]
            
            return {
                'version': version,
                'type': tx_type,
                'property_id': property_id,
                'amount': amount
            }
    
    return None
```

**Step 4: Verification**

You can use Omni Explorer to view:

- https://omniexplorer.info/
- Enter transaction ID to view Omni layer parsing results

**Using Complete Code Implementation:**

The above code snippets show core logic. Complete implementation (including error handling, type checking, etc.) is located in the `code/omni/` directory.

**Key Understanding:**

What Bitcoin nodes see:

```
- A normal transaction with an OP_RETURN output
- Transferred 546 sats to some address
- Other Bitcoin change
```

What Omni nodes see:

```
- A USDT transfer
- From address A to address B
- Amount 10.00000000 USDT
- Property ID 31 (USDT)
```

**This is the essence of the "two-layer protocol":**

- Bottom layer: Bitcoin ensures transaction immutability
- Upper layer: Omni interprets the meaning of data

---

## 4.6 How OP_RETURN "Changed" Bitcoin

Omni's use of OP_RETURN was so successful
that Bitcoin Core later formally supported it in policy:

- OP_RETURN is the recommended data entry point
- Safest, least disruptive to consensus
- Does not occupy UTXO
- Easily identifiable by wallets and nodes
- Can be pruned in the future

From then on, the "standard entry point" for Bitcoin data embedding was established:

**Write explicit data → Use OP_RETURN.**

This laid the foundation for:

- Counterparty
- RSK
- Runes
- Taproot Assets (partial use)
- All metadata projects on Bitcoin

---

## 4.7 Omni's Success and Failure

### Successes

- Pioneered the first generation of explicit on-chain data protocols
- USDT successfully ran on Bitcoin for many years
- First to implement on-chain assets, DEX, etc.
- Created the "parser + state machine" architecture
- Community widely adopted OP_RETURN (de facto standard)

### Failures

- Payload capacity limited (40-80 bytes)
- Full state on-chain → permanent bloat
- Relies on full node scanning, difficult to lighten
- Complex asset logic, state machine prone to errors
- Insufficient to express complex contract logic
- Completely replaced by Ethereum (general asset functionality)
- Later lost advantages to Runes, RGB

Ultimately, Omni evolved into:

**A single-purpose system: USDT on Bitcoin.**

Its general asset system functionality gradually declined.

---

## 4.7.5 USDT: Omni's Greatest Success (and Only Surviving Application)

### Historical Background

- **October 2014**, Tether issued USDT based on Omni (Property ID: 31)
- **February 2015**, first USDT transaction on-chain
- Became the first large-scale adoption of a Bitcoin asset protocol application

### Technical Implementation

USDT uses Omni Simple Send (Transaction Type 0):

```
OP_RETURN 6f6d6e69 00000000 0000001f 000000003b9aca00
          ^^^^^^   ^^^^^^^^ ^^^^^^^^ ^^^^^^^^^^^^^^^^
          "omni"   version  type=0   amount (decimal)
                            prop=31
```

**Meaning of Property ID 31:**

- Property ID is the asset identifier in the Omni protocol
- USDT = Property 31
- All USDT transfers must include this ID

### Transfer Process

**1. On-Chain Part:**

```
Input: Sender's Bitcoin UTXO
Output 0: OP_RETURN (Omni instruction)
Output 1: Receiver address (dust amount, typically 546 sats)
Output 2: Change address (remaining Bitcoin)
```

**2. Off-Chain State Machine:**

```
Omni nodes scan blockchain
  ↓
Identify "omni" magic bytes in OP_RETURN
  ↓
Parse Property ID and amount
  ↓
Update sender and receiver USDT balances
```

### Real-World Case

**Transaction:** `21c3354ab4422faca43761826884bb4398871a2c404a2fff71be91c9502d3ba0`

This is a transaction transferring 10.00000000 USDT:

- **Bitcoin layer**: Transferred thousands of sats
- **Omni layer**: Transferred 10 USDT
- Two state machines run in parallel, without interference

### Why Did USDT Choose Omni?

Selection context in 2014-2015:

- Ethereum had not yet launched mainnet (July 2015)
- Bitcoin was the most mature blockchain
- Omni was the only mature Bitcoin asset protocol at the time
- OP_RETURN provided a relatively safe data embedding method

### Why Did USDT Later Migrate to Ethereum?

- High Bitcoin transaction fees (during 2017 bull market)
- Ethereum ERC-20 more flexible (smart contracts)
- On-chain confirmation time (10 minutes vs 15 seconds)
- Better programming capabilities and ecosystem

### Current Status (2025)

- USDT on Omni still runs, but transaction volume is small
- Most USDT is on Ethereum, Tron, and other chains
- Omni protocol has essentially become synonymous with "USDT on Bitcoin"
- The original grand vision of a "second-layer financial system" has disappeared

---

## 4.8 Omni's Position in Bitcoin Data Embedding History

### Technical Dimension

**Omni's Three "Firsts":**

1. **First protocol to use OP_RETURN at scale**
   - Promoted Bitcoin Core's formal support for OP_RETURN
   - Established the paradigm of "OP_RETURN = data channel"

2. **First on-chain instruction + off-chain state machine architecture**
   - Later copied by Counterparty, Runes, BRC-20
   - Proved the feasibility of "minimal on-chain commitment"

3. **First successful Bitcoin asset protocol**
   - USDT has run for over 10 years
   - Processed tens of millions of transactions

### Historical Dimension

Technology evolution timeline:

```
2012  Colored Coins    → Zero data, pure interpretation (failed)
2013  Mastercoin       → Multisig embedding (UTXO pollution)
2014  OP_RETURN        → Bitcoin Core provides formal channel
2014  Omni             → OP_RETURN standardization
2014  Counterparty      → Pushed to limits (OP_RETURN + multisig)
2017  SegWit           → Witness emergence
2021  Taproot          → Merkle tree structure
2023  Ordinals         → Witness + Taproot (new paradigm)
```

### Philosophical Dimension

**Omni proved an important point:**

"Bitcoin can carry complex asset systems without modifying the consensus layer."

This point influenced the entire Bitcoin L2 ecosystem:

- Don't fork Bitcoin
- Innovate within existing rules
- Extend functionality with "interpretation layer"

### Limitations

**Omni also exposed fundamental problems of first-generation protocols:**

1. **Full state scanning** → Cannot be lightened
2. **No on-chain verification** → Relies on client consensus
3. **Capacity limited** → 80 bytes limits expressiveness
4. **Single-purpose evolution** → Ultimately only USDT remains

**These problems drove the evolution of next-generation protocols:**

- Counterparty: Break capacity limits
- Stamps: Pursue data non-prunability
- Ordinals: Leverage Witness discount
- RGB: Completely off-chain state

### Position in This Book's Structure

Colored Coins belongs to:

**Generation 0: Implicit protocol (no bytes written)**

Omni belongs to:

**Generation 1: Explicit OP_RETURN protocol (fixed-length payload written)**

In data embedding history, Omni's significance lies in:

- First use of the "proper way" to write data
- Established on-chain instruction paradigm
- Made on-chain data writing mainstream
- Made assets a reality in the Bitcoin ecosystem
- Influenced all subsequent protocols (BRC-20 / Runes / RGB)

In this book's structure, Omni is the key bridge connecting Colored Coins → Counterparty / Stamp.

---

## 4.9 Summary: Omni's Engineering Significance

Omni represents:

- ✔ "Explicit data embedding" first moving toward standardization
- ✔ OP_RETURN becoming Bitcoin's structural data entry point
- ✔ On-chain state machine + instruction-based protocol model
- ✔ First-generation formal implementation of asset protocols
- ✔ Critical foundation for USDT to run on Bitcoin

And points the direction for subsequent development:

**Next-generation protocols must write more data, have stronger structure, and greater expressiveness.**

---

## 4.10 Why Counterparty? Omni's Limitations

Although Omni successfully ran USDT, its design has several fundamental limitations:

### 1. Insufficient Data Capacity

- OP_RETURN limited to 80 bytes
- Cannot express complex DEX orders, crowdsale parameters
- Limits protocol expressiveness

### 2. Functional Simplification

- Original grand vision (DEX, contracts, crowdsale) largely failed
- Only "issuance" and "transfer" were widely used in practice
- Complex functions difficult to implement within 80 bytes

### 3. Cannot Encode Complex Data Structures

- 80 bytes difficult to carry structured data
- Cannot implement on-chain orderbook
- Cannot encode images, documents, etc.

### Counterparty's Improvements

- Break 80-byte limit (using multisig)
- Implemented true DEX
- Supports more complex asset types
- Can encode arbitrary binary data

**This leads to the next chapter:**

**Chapter 5: Counterparty — OP_RETURN + Multisig Structured Payload**

How does Counterparty combine "OP_RETURN + multisig"
to break Omni's capacity limitations and build a more powerful asset system?

Counterparty directly pushed "data embedding" to the extreme, even stuffing data into bare multisig public keys.
It is the predecessor of Stamp and Ordinals.

---

**💡 Code Implementation for This Chapter:** Complete runnable code is located in the `code/omni/` directory. See README.md in that directory for details.

