# How People Tried to Push Non-Transaction Data into Bitcoin

*A Reproducible Engineering History of All Data-Embedding Techniques on Bitcoin*

> **Status:** Early draft — chapters are being written and code examples are coming online. Feedback and issues are very welcome.

**中文版 / [Chinese Version](book/translation/zh/home_cn.md)**

---

## Overview

This repository documents — systematically, historically, and with reproducible engineering detail — how people have attempted to embed non-transaction data into Bitcoin over the past fifteen years.

Bitcoin was not designed as a general-purpose data layer.
Yet developers, protocol designers, artists, and financial engineers have continuously attempted to store:

- metadata
- images
- instructions and scripts
- asset state
- Merkle commitments
- arbitrary binary data

…within Bitcoin's rigid UTXO and script model.

This project is the first systematic attempt to:

- Reconstruct every major data-embedding technique ever used on Bitcoin
- Explain how each works structurally
- Provide reproducible testnet implementations for verification

---

## Motivation

Bitcoin's consensus rules do not expose an explicit "data field."
However, its structure contains implicit seams where data can be injected:

- **Inputs** (scriptSig, witness, script path)
- **Outputs** (scriptPubKey, OP_RETURN, Taproot leaf commitments)

Every protocol — Colored Coins, Omni, Counterparty, Stamps, Ordinals, Atomicals, Runes, RGB — is fundamentally a different way of exploiting these structural seams.

This repository aims to:

- Clarify consensus vs. policy boundaries
- Demonstrate what Bitcoin actually allows, not what people assume
- Document all known engineering patterns for embedding data
- Provide reproducible code so readers can run protocols on testnet
- Give protocol researchers a clean reference to avoid repeating past mistakes
- Help the Bitcoin developer community evaluate future proposals in context

**This is not a promotional document.**
**It is a technical history, reconstructed from first principles.**

---

## Repository Structure

```
bitcoin-data-embedding/
│
├── book/
│   ├── manuscript/
│   │   ├── ch01_intro.md
│   │   ├── ch02_data-entry-map.md
│   │   ├── ch03_colored-coins.md
│   │   ├── ch04_omni-opreturn.md
│   │   ├── ch05_counterparty-multisig.md
│   │   ├── ch06_stamp-bare-multisig.md
│   │   ├── ch07_p2wsh-data-container.md
│   │   ├── ch08_ordinals-witness.md
│   │   ├── ch09_atomicals-taproot.md
│   │   ├── ch10_runes-opreturn2.md
│   │   ├── ch11_rgb-client-validation.md
│   │   └── ch12_future-directions.md
│   │
│   └── translation/
│       └── zh/
│           ├── ch01_intro_cn.md
│           ├── ch02_data-entry-map_cn.md
│           └── ...
│
├── code/
│   ├── colored-coins/
│   ├── omni/
│   ├── counterparty/
│   ├── stamps/
│   ├── p2wsh/
│   ├── ordinals/
│   ├── atomicals/
│   ├── runes/
│   └── rgb/
│
├── diagrams/
│
├── tools/
│   ├── witness_dump.py
│   ├── taproot_merkle_builder.py
│   └── script_visualizer.py
│
└── README.md
```

---

## Content Summary

Each chapter explains:

- Historical context and motivation
- Exact engineering mechanism
- Raw transaction breakdown
- Witness / script path analysis
- How the protocol fits into Bitcoin's data model
- Failure modes and long-term constraints
- What later protocols inherited or corrected

### Chapter List

1. **Preface** — Why people wanted to store non-transaction data on Bitcoin
2. **A Complete Map of Data Entry Points** — Input vs output structure analysis
3. **Colored Coins** — Zero-byte protocols and implicit state machines
4. **Omni (Mastercoin)** — OP_RETURN's first formal use
5. **Counterparty** — Embedding data via OP_RETURN + multisig techniques
6. **Stamps** — Extreme bare-multisig data payloads
7. **P2WSH** — When scripts became structured data containers
8. **Ordinals** — Arbitrary witness data and "inscriptions"
9. **Atomicals** — Taproot-based object encoding
10. **Runes** — A modern OP_RETURN asset model
11. **RGB** — Client-side validation and off-chain state commitment
12. **Future Directions** — Commitments, covenants, and data-layer boundaries

Each chapter includes testnet transactions, raw hex dumps, and reproducible code.

---

## Reproducibility

All examples in the book are fully reproducible using:

- Python 3
- `bitcoinlib` / `bitcoinutils`
- Bitcoin Core (regtest or testnet)
- Simple command-line scripts

> **Note:** The `code/` and `tools/` directories are being filled in chapter by chapter. In early drafts, some paths may not exist yet.

### Quick Start

```bash
cd code/omni/
python3 create_opreturn_tx.py
```

or

```bash
cd code/ordinals/
python3 inscribe.py
```

Each code directory includes:

- `README.md` with setup instructions
- Raw example transactions
- Annotated witness/script dumps
- Diagrams explaining data placement

---

## Design Principles

This project is built on four core principles:

### 1. Reproducibility

Every claim is supported by real raw transactions that can be verified on-chain.

### 2. Neutrality

No protocol is promoted or criticized.
Each is evaluated strictly on engineering merit and technical accuracy.

### 3. Boundary Clarity

Clear distinction between:

- Consensus rules (what Bitcoin nodes must accept)
- Policy rules (what Bitcoin Core chooses to relay)
- Wallet behavior (implementation-specific)
- Interpretation layers (protocol-specific parsing)

### 4. Minimal Assumptions

All explanations derive from Bitcoin's actual data structure, not speculative narratives or marketing claims.

---

## Contributing

Pull requests and issues are welcome.

We particularly appreciate:

- Corrections to historical details and timelines
- Additional testnet examples with raw transaction hex
- Script/path breakdowns and visualizations
- Diagrams or visual aids
- Policy and historical context from long-time Bitcoin contributors

This is intended as a collaborative technical reference for the Bitcoin developer community.

---

## License

- **Book text**: Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)
- **Code**: MIT License

---

## Additional Formats

This project is also available in:

- **GitBook** — Visual reading experience with interactive diagrams
- **Medium Series** — Chapter-by-chapter articles
- **Leanpub Book** — PDF/ePub formats (free or pay-what-you-want)

Links will be added here when each format goes live.

---

## Contact

For discussion, corrections, or collaboration:

- **Twitter**: [@aaron_recompile](https://twitter.com/aaron_recompile)
- **GitHub Issues**: For technical questions and corrections

---

## Final Note

Bitcoin's history of data embedding is not a curiosity —
it is essential to understanding:

- Protocol design constraints
- Policy debates and their technical roots
- Fee market evolution
- The future of Bitcoin as a programmable settlement layer

This repository documents that history from the viewpoint of a builder, not a speculator.

**Every protocol discussed here represents a real attempt to push Bitcoin's boundaries.**
**Understanding how they work — and why some succeeded while others failed — is crucial for anyone building on Bitcoin.**
