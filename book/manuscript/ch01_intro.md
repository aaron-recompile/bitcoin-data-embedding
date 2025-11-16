# Preface

## How People Tried to Push Non-Transaction Data into Bitcoin

### —— A Technical and Reproducible History

From Bitcoin's very first day, people have continuously attempted to write data onto the blockchain that was never intended for transaction validation. Text, images, instructions, asset states, structured metadata—these contents do not belong to the core purpose of the UTXO model, yet they have persistently found their way into the gaps between inputs and outputs through various means.

Bitcoin's consensus layer cares about only two things:

1. Whether inputs satisfy spending conditions (input - scriptSig or Witness)
2. Whether outputs are correctly locked (output - scriptPubKey)

Any information beyond this, if it can be embedded and interpreted, exists not because the system was designed with space for it, but because some structural element has exploitable margins.

The central theme of this book is to systematically answer one question:

"How have people, over the past fifteen years, continuously exploited Bitcoin's most primitive and strict structures to write non-transaction data into the system?"

---

## Two Eternal Vectors: Input and Output

Regardless of what protocols are called, regardless of how outer-layer standards are designed, their essence remains the same:

- Either write into inputs (Witness / Taproot leaf)
  Leveraging the flexibility of witness fields to carry arbitrary bytes.
- Or write into outputs (OP_RETURN / scriptPubKey)
  Embedding data hashes, structural fragments, or instructions into locking scripts.

All "protocols," all "assets," all "NFTs," and all "on-chain state machines" depend on these two fundamental vectors for their existence.

---

## Two Forms of Data Expression: NFTs and Structured Assets

The data people write can be broadly divided into two categories:

1. **Display-oriented data**
   Images, text, objects—what we call NFTs.
   At its core, this is content that can be rendered by external interpreters.

2. **State-oriented data**
   Asset balances, ownership, ABIs, state transition instructions.
   These constitute an "off-chain state machine," while Bitcoin stores only its minimal mapping.

From Colored Coins to Omni, Counterparty, Stamps, Ordinals, Atomicals, and RGB,
the divergence among these protocols is not in philosophy, but in how they exploit Bitcoin's smallest structural units to express their state machines.

---

## Why Study Them?

Bitcoin Core developers do not endorse these practices.
Many embedding methods are viewed as policy-level spam, not intended protocol features.

Yet from an engineering perspective, these attempts have irreplaceable value:

- They expose the true boundaries of Bitcoin's structure
- They demonstrate the expressive power of scripts, witness, and Taproot
- They force us to understand the distinction between consensus and policy
- They represent all possibilities and all limitations for decentralized applications on Bitcoin

This book will use reproducible testnet experiments, verifiable scripts, raw transaction parsing, witness dumps, and Taproot Merkle construction
to explain, one by one, the principles, advantages, vulnerabilities, and engineering trade-offs of these protocols.

---

## The Book's Approach

This is not a book that encourages or opposes any protocol.
Nor is it market analysis or a speculation guide.

This is a book that dissects the history of Bitcoin data embedding from the ground up, using engineering methods.

We will:

- Systematically reconstruct all historical embedding methods
- Conduct reproducible experiments on all protocols
- Clarify the boundaries between consensus / policy / application
- Abstract data writing models from actual code and on-chain behavior
- Use real logic to explain why some protocols survive while others inevitably fail

When you finish this book, you will not only understand "what these protocols are,"
you will understand "why they exist, why they do what they do, and why they work or don't work."

---

## Intended Audience

- Bitcoin script and Taproot engineers
- Bitcoin Core / Lightning / Wallet developers
- Researchers building next-generation on-chain protocols
- Those who want to fundamentally understand asset protocols and NFT design
- Any engineer who wants to master "how data flows in Bitcoin"

---

## Conclusion: Bitcoin Is Not an Application Chain, But Applications Will Not Disappear

As long as Bitcoin allows you to embed data in inputs or outputs—
whether script, witness, Taproot leaf, or future covenants—
people will continue attempting to build assets, content, and state machines on top of it.

This is not a deviation from Bitcoin, but an exploration of its boundaries.

This book attempts to systematically describe these boundaries for the first time, using reproducible engineering.

