# Chapter 9 —— Runes —— Modern OP_RETURN "Return"

## Runes —— Modern OP_RETURN Return

### From Witness Complexity to OP_RETURN Simplicity

In Chapter 8, we saw that Ordinals and Atomicals pushed witness and Taproot capabilities to the extreme,
spawning token protocols like BRC-20, but also exposing problems:

- Token state completely depends on off-chain indexer parsing
- Different indexers may produce different results
- Disconnected from Bitcoin's native UTXO model
- Requires complex indexing infrastructure

In April 2024 (Bitcoin's fourth halving), Ordinals creator Casey Rodarmor launched the **Runes** protocol,
choosing an almost "anti-climactic" path:

**No longer abusing scripts, no longer stuffing complex structures, but returning to the simple OP_RETURN pattern.**

It can be viewed as:

**A "modern refactoring" of Omni:**
- Simpler structure
- Token-only focus
- Easier to parse
- Token state directly bound to UTXO
- No complex indexer required

This chapter will analyze:

- Why Runes is a "correction" to BRC-20 chaos
- How it returns to the OP_RETURN route
- How the UTXO binding mechanism works
- Its symbolic significance in data embedding history

---

## 9.1 Runes' Background: Response to BRC-20 Chaos

### 9.1.1 BRC-20's Problems

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

**Core Problems:**

1. **Complete dependence on off-chain indexers**: Token state must be rebuilt by scanning all inscription transactions
2. **Indexer divergence**: Different indexers may produce different balance results
3. **Disconnected from UTXO**: Token transfers don't follow Bitcoin's UTXO rules
4. **Complex parsing**: Requires parsing JSON, handling inscription order, maintaining state databases

### 9.1.2 Runes' Design Philosophy

Runes' philosophy is extremely restrained compared to previous protocols:

- Doesn't attempt to build a full-featured platform
- Doesn't create complex contracts
- Doesn't build on-chain object trees
- **Focuses on placing a simple instruction in OP_RETURN:**
  "Who owns how much of which Rune."

The main differences from Omni are:

- Simpler design, more fixed fields
- Unified specification for symbols, names, IDs
- **Token state directly bound to UTXO** (key innovation)
- Compatibility considerations for modern Bitcoin ecosystem
- Attempts to reduce parsing and implementation burden

**In other words:**

Runes is a modern rewrite of "OP_RETURN as token protocol," not a completely new invention.
It draws from Omni's "event log" pattern and introduces UTXO binding mechanism.

---

## 9.2 Runes' Core: UTXO-Bound Token Model

### 9.2.1 UTXO Binding Mechanism

Runes' key innovation is **directly binding token state to UTXO**:

- Token balances are stored in specific UTXOs
- Transfers are spending UTXOs containing tokens, creating new UTXOs
- **Conceptually no longer depends on a global, complex off-chain state machine, only needs indexing at the UTXO level**

**Important Clarification:**

Although Runes binds balances to UTXO, calculating current balances still requires indexers to scan historical runestones (messages in OP_RETURN), applying edicts (allocation instructions) to rebuild state.
Bitcoin Core nodes don't natively support Runes queries; indexers (like ord) must be used to process serialized edicts.

**Differences from BRC-20:**

- Runes' indexing model is closer to UTXO, simpler in itself
- Doesn't need to maintain complex event replay state machines
- Doesn't need to parse JSON and handle inscription order
- Under the same version of rules, all implementations should produce consistent parsing results for the same chain data

Fundamental differences from BRC-20:

| Dimension | BRC-20 | Runes |
|-----------|--------|-------|
| State storage | Off-chain indexer maintains global state | **In UTXO, but requires indexer calculation** |
| Transfer logic | Parse JSON instructions | **Spend UTXO + edicts** |
| Balance query | Scan all inscription JSON | **Scan runestones, index by UTXO** |
| Indexer | Required (complex state machine) | **Required (but simpler model)** |

### 9.2.2 Why Choose OP_RETURN?

Runes chooses OP_RETURN over witness for reasons including:

1. **Simplicity**: OP_RETURN is Bitcoin's most direct data entry point
2. **Visibility**: Data is permanently visible upon creation, no commit-reveal needed
3. **UTXO compatibility**: OP_RETURN outputs, though unspendable, can be combined with other outputs
4. **Policy stability**: OP_RETURN is a data channel explicitly supported by Bitcoin Core (Note: From a 2025 perspective, Bitcoin Core's OP_RETURN policy is relatively stable with no major changes)
5. **Sufficient capacity**: In current standards, OP_RETURN payload space is relatively limited, but sufficient for compactly encoded instructions like Runes

**Comparison with witness:**

| Feature | Witness (Ordinals/BRC-20) | OP_RETURN (Runes) |
|---------|---------------------------|-------------------|
| Cost | 1 WU/byte (75% discount) | 4 WU/byte |
| Visibility | Requires commit-reveal | Immediately visible |
| UTXO binding | Difficult | **Direct** |
| Parsing complexity | High (requires script parsing) | Low (direct read) |

---

## 9.3 Runes' Structural Features: Short, Small, Fixed, Easy to Parse

### 9.3.1 Data Format

A Rune operation on-chain primarily appears as:

```
OP_RETURN <payload>
```

These OP_RETURN messages are collectively called **Runestone** in the protocol.

**Three Types of Runestone Messages:**

- **Etch (Create)**: Create new Rune tokens
- **Transfer**: Allocate tokens to output UTXOs through edicts
- **Burn**: Destroy tokens

Payload structure (simplified illustration):

| Field | Length | Meaning |
|-------|--------|---------|
| Protocol header | 4 bytes | Rune protocol marker (ASCII 'R') |
| Operation type | Variable | Etch / Transfer / Burn |
| Asset identifier | Variable | Rune ID or name (LEB128 encoded) |
| Edicts | Variable | Allocation instruction list (rune ID, amount, output index) |
| Optional fields | Variable | Other parameters |

**Technical Details:**

Runestone uses LEB128 encoding varints (128-bit integer sequences) to compress data.
If payload is invalid, the "cenotaph" mechanism is triggered (burning input runes).
This mechanism prevents invalid transactions from abusing the protocol, increasing protocol robustness:
Invalid runestones don't cause tokens to be lost to unknown addresses, but are explicitly burned, avoiding state confusion.

**Key Points:**

- No script abuse
- No multisig data payloads introduced
- No attempt to encode all complex state on-chain
- Compression of parsing rules into a relatively stable, predictable format

### 9.3.2 Format Comparison with Omni

| Dimension | Omni | Runes |
|-----------|------|-------|
| Protocol header | 4 bytes ("omni") | 4 bytes (Rune marker) |
| Version number | 2 bytes | Simplified (may be omitted) |
| Instruction type | 2 bytes | 1-2 bytes |
| Field design | Flexible but complex | **Fixed and simple** |
| Extensibility | Supports multiple asset types | **Token-focused** |

---

## 9.4 Comparison with Omni: From "Swiss Army Knife Platform" to "Dedicated Simplified Protocol"

### 9.4.1 Design Goal Comparison

| Protocol | Design Goal | Complexity | Expression Range |
|----------|-------------|------------|----------------------|
| Omni | Universal asset layer + DEX | High | Broad, but high implementation cost |
| Runes | Token protocol focused | Medium-low | Narrow but clear |

### 9.4.2 Architecture Pattern Comparison

**Omni Pattern (Event Log):**

```
On-chain: OP_RETURN instructions (event log)
Off-chain: State machine scans all instructions, rebuilds current state
```

**Runes Pattern (UTXO Binding):**

```
On-chain: OP_RETURN runestone (edicts) + UTXO binding
Off-chain: Indexer scans runestones, builds index by UTXO dimension (simpler than Omni)
```

### 9.4.3 Significance from "Data Embedding" Perspective

Runes means engineers began to reflect:

**"Do we really need a super-protocol with everything on-chain?"**

Instead:

- Return to simple models
- Delegate more logic to clients
- **Utilize UTXO model's inherent state capabilities**
- Maintain OP_RETURN's simplicity and auditability

---

## 9.5 Technical Comparison: Runes vs BRC-20

### 9.5.1 State Management Comparison

**BRC-20:**

```
1. Scan all Ordinals inscriptions
2. Parse JSON instructions
3. Maintain off-chain state database
4. Different indexers may produce different results
```

**Runes:**

```
1. Indexer scans historical runestones (OP_RETURN messages)
2. Apply edicts to calculate each UTXO's balance
3. State bound to UTXO, but requires indexer calculation
4. Under the same version of rules, parsing results should be consistent
```

### 9.5.2 Transfer Process Comparison

**BRC-20 Transfer:**

```
1. Create new inscription (JSON transfer instruction)
2. Indexer parses instruction
3. Update off-chain state database
4. Wallet queries indexer for balance
```

**Runes Transfer:**

```
1. Spend UTXO containing tokens
2. Write runestone (containing edicts) in OP_RETURN
3. Edicts specify which output UTXOs tokens are allocated to
4. Indexer scans runestone, updates UTXO balance index
5. Wallet queries indexer for balance (or runs own indexer)
```

### 9.5.3 Wallet Implementation Complexity

| Dimension | BRC-20 | Runes |
|-----------|--------|-------|
| Required components | Indexer + complex state database | **Indexer + UTXO index** |
| Sync cost | Scan all historical inscription JSON | **Scan historical runestones, index by UTXO** |
| Implementation difficulty | High (JSON parsing, state machine) | **Medium-low (fixed format, UTXO model)** |
| Third-party dependency | Yes (complex indexer) | **Yes (but simpler model, can self-build)** |

---

## 9.6 Position in Data Embedding History: A "Return" and "Convergence"

### 9.6.1 Historical Evolution Path

```
Colored Coins (implicit state)
  ↓
Omni (OP_RETURN event log)
  ↓
Counterparty (OP_RETURN + multisig dual channel)
  ↓
Stamps (bare multisig extremization)
  ↓
P2WSH (witness structured container)
  ↓
Ordinals/Atomicals (witness data explosion)
  ↓
BRC-20 (witness JSON tokens, indexer dependent)
  ↓
**Runes (return to OP_RETURN, UTXO binding)**
```

### 9.6.2 Runes' Symbolic Significance

Runes marks:

- After witness + Taproot were extensively used for content/object carrying
- A community tendency emerged:
- **Push complexity back off-chain, make on-chain token protocols as simple as possible**

From the "non-transaction data history" perspective, it's like:

**"A modern rewrite of first-generation Omni"**

After experiencing Colored Coins → Omni → Counterparty → Stamps → P2WSH → Ordinals → Atomicals,
a "subtraction design" of the OP_RETURN route.

### 9.6.3 Position in Data Embedding History

| Era | Protocol | Data Location | Characteristics | State Management |
|-----|----------|---------------|-----------------|------------------|
| Early | Omni | OP_RETURN | Event log | Off-chain state machine |
| Mid | Counterparty | OP_RETURN + multisig | Dual channel | Off-chain state machine |
| SegWit | P2WSH | Witness | Structured container | Off-chain parsing |
| Taproot | Ordinals/BRC-20 | Witness | Data explosion | **Off-chain indexer** |
| **Modern** | **Runes** | **OP_RETURN** | **Simplified return** | **UTXO binding** |

**Key Insight:**

Runes is not a technological regression, but design maturity.
It proves: **Simple, verifiable designs consistent with Bitcoin's native model are more valuable than complex feature stacking.**

---

## 9.7 Summary: Modernization of the OP_RETURN Route

Runes demonstrates an important trend:

**After experiencing the complexity of the witness era,**
**engineers began returning to OP_RETURN's simplicity,**
**and utilizing UTXO model's inherent capabilities.**

**Engineering Significance:**

- OP_RETURN remains Bitcoin's most direct, simplest data entry point
- UTXO binding makes token state model closer to Bitcoin's native model, simpler indexing
- Although indexers are still needed, no longer need to maintain complex event replay state machines
- Design lowers implementation barriers, improves verifiability

**Design Philosophy:**

- Not all functionality needs to be implemented on-chain
- Utilizing Bitcoin's native model (UTXO) is more reliable than creating new models
- Auditable designs are more suitable for long-term evolution than complex feature stacking

**Position in Data Embedding History:**

Runes marks the modern return of the OP_RETURN route.
It's not a negation of the witness route, but demonstrates:
**Different data embedding methods suit different use cases.**
**For token protocols, simple, UTXO-native designs may be more suitable than complex witness structures.**

Next chapter, we'll truly shift perspective to the pinnacle of "off-chain state machine + on-chain commitment" — RGB.

RGB represents another design philosophy:
**Minimize on-chain data, maximize off-chain validation,**
**guarantee state consistency through cryptographic commitments.**

