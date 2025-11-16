# Chapter 2

## 比特币数据入口全图：Input vs Output 的完整结构

### The Complete Map of Where Data Can and Cannot Enter Bitcoin

上一章说明了一个核心事实：

比特币只保留两类字节：输入数据（用于解锁）与输出数据（用于上锁）。
所有非交易数据的写入，都只能沿着这两个方向展开。

但这句话本身仍然过于抽象。
本章将逐层拆开实际的比特币交易结构，明确：
哪些字段可以承载字节、承载多少、有什么规则、哪些是共识 vs 策略、哪些可以滥用、哪些不可滥用。

**更重要的是，我们需要理解一个关键区别：**

- **数据在 Output**：数据创建时就永久可见，直接存储在 UTXO 集中
- **数据在 Input Witness**：数据在解锁时才揭示，通过 witness 字段承载

这个区别决定了数据的经济成本、可见性和协议设计选择。

本章的内容可以直接作为一个"数据嵌入完整地图"使用，帮助理解所有协议的底层技术基础。

---

## 2.1 比特币交易的字节结构总览

### 2.1.1 交易四大组成部分

比特币的交易（Transaction）由四大块组成：

```
Version
Inputs[]
Outputs[]
Locktime
```

其中**只有 Inputs 与 Outputs 包含"可插入字节"的空间**。

但需要明确：**数据实际存储的位置**与**创建数据结构的入口**可能不同。

### 2.1.2 交易结构可视化

在深入讨论各个数据入口之前，我们先建立一个完整的交易结构图景。

**交易头部：**

```
┌─────────────────────────────────────────┐
│ Version (4 bytes)                        │
│ ─────────────────────────────────────── │
│ Inputs Count (VarInt)                    │
│ ─────────────────────────────────────── │
│ Inputs[]                                 │
│ ─────────────────────────────────────── │
│ Outputs Count (VarInt)                  │
│ ─────────────────────────────────────── │
│ Outputs[]                                │
│ ─────────────────────────────────────── │
│ Locktime (4 bytes)                      │
└─────────────────────────────────────────┘
```

**Inputs 结构：**

```
┌─────────────────────────────────────────┐
│ Input #0                                │
│ ├─ Previous TXID (32 bytes)            │
│ ├─ Previous Vout (4 bytes)             │
│ ├─ scriptSig Length (VarInt)           │
│ ├─ scriptSig (📝 可嵌入数据 - Legacy)  │
│ ├─ sequence (4 bytes)                  │
│ └─ witness (📝 可嵌入数据 - SegWit)    │
│    └─ [element_0, element_1, ...]      │
│                                         │
│ Input #1                                │
│ └─ ...                                  │
└─────────────────────────────────────────┘
```

**Outputs 结构：**

```
┌─────────────────────────────────────────┐
│ Output #0                               │
│ ├─ Value (8 bytes)                     │
│ └─ scriptPubKey Length (VarInt)        │
│    └─ scriptPubKey                     │
│       ├─ OP_RETURN <data> (📝 方式1)   │
│       ├─ Fake Multisig (📝 方式2)     │
│       └─ 特殊脚本结构 (📝 方式3)      │
│                                         │
│ Output #1                               │
│ └─ ...                                  │
└─────────────────────────────────────────┘
```

### 2.1.3 数据入口标识说明

**关键标记：**
- 📝 表示可以嵌入任意数据的位置
- Legacy 交易使用 scriptSig，SegWit/Taproot 交易使用 witness
- Output 数据入口：OP_RETURN、Fake Multisig、特殊 scriptPubKey

**关键区别：**
- **数据在 Output**：数据创建时就永久可见，直接存储在 UTXO 集中
- **数据在 Input Witness**：数据在解锁时才揭示，通过 witness 字段承载，享受 75% 成本折扣

接下来的章节，我们将逐一剖析这些数据入口。

---

## 2.2 Input 的数据入口

一个 Input 简化结构如下：

```
{
  prevout: <txid:vout_index>
  scriptSig: <arbitrary bytes>   # legacy
  sequence: <4 bytes>
  witness: [stack elements]      # segwit
}
```

**Input 内可写入数据的字段有两个：**

1. **scriptSig**（legacy）
2. **witness**（segwit）

而它们的能力差别极大。

### 2.2.1 scriptSig —— 旧时代"可插入字节"的入口（已几乎被废弃）

scriptSig 本质上是：

```
<push> <signature>
<push> <public key>
```

但在合法性要求之外，它曾经可以塞任意字节。
这导致早期出现过：

- 把数据当成"假脚本"塞进去
- 把私有数据放进去（极其浪费）
- 用花费路径编码状态（彩色币模式）

但是：

- ❌ 2017 以后，这条通道几乎被淘汰
  - scriptSig 会影响 TXID
  - 容易引入 malleability
  - mempool policy 不鼓励把脚本当成数据箱

因此：scriptSig 是"历史性的嵌入路径"，但仍然是数据入口之一。

### 2.2.1 scriptSig 示例

早期的彩色币协议（如 Colored Coins）会在 scriptSig 中嵌入数据。

**历史背景：**
- 2012-2014 年间，部分彩色币实现尝试在 scriptSig 中编码资产信息
- 由于 scriptSig 会影响 TXID，这种方式容易导致交易可塑性（malleability）问题
- 2017 年 SegWit 激活后，scriptSig 数据嵌入方式基本被废弃

**现状：**
- 现在已很少使用 scriptSig 作为数据载体
- 现代协议主要使用 witness 或 OP_RETURN
- 详见第 3 章关于彩色币的完整历史

### 2.2.2 witness —— 比特币史上最强大的数据载体

witness 出现后，数据嵌入进入"现代时代"。

Witness 的结构是：

```
witness: [ element_0, element_1, ..., element_n ]
```

每一个 witness 元素（stack item）都是：

- 任意字节长度（受 block weight 限制）
- 不影响 TXID
- 不进入 script 执行逻辑（仅被 push）
- 必须符合 stack push 规则
- 不必有逻辑意义（只要脚本通过验证）

**Witness discount 机制（BIP 141）：**

- Base transaction data: 4 weight units per byte
- Witness data: 1 weight unit per byte
- Block weight 上限: 4,000,000 units (约等于4MB)

这意味着：

- Witness 数据相比传统数据享有 **75% 的成本折扣**
- 理论上可以在一个区块内塞入接近 4MB 的 witness 数据
- 这是为什么 Ordinals、Atomicals 选择 witness 的经济学原因

这一点极其重要：

**Witness 中的每个元素不需要"有意义"，它只需要被压栈，然后在脚本执行过程中被忽略即可。**

这为后来的：

- Ordinals（图片）
- BRC-20（结构化 JSON）
- Atomicals（payload）
- Taproot script-path 嵌入
- 以及任何类型的二进制数据

提供了极大自由度。

Witness 的限制主要来自两条：

1. Block weight（软限制但重要）
2. 脚本必须执行成功（但可设计成一定成功）

这就是为什么 Ordinals 和 Atomicals 能装几 MB 的内容。

**Witness 是真正的"现代数据入口"。**

### 2.2.3 实战案例：Ordinals 铭文交易

**交易：** `6fb976ab49dcec017f1e201e84395983204ae1a7c2abf7ced0a85d692e442799`

[Mempool 链接](https://mempool.space/tx/6fb976ab49dcec017f1e201e84395983204ae1a7c2abf7ced0a85d692e442799)

这是一个典型的 Ordinals 铭文交易，将 PNG 图片刻在比特币上。

**Witness 结构：**

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
      OP_PUSHDATA2 [完整PNG数据 ~2KB]
      OP_ENDIF

[2] Control Block: c04a3ca2cf35f7902df1215f...
```

**关键点：**

- 数据被包裹在 `OP_FALSE OP_IF ... OP_ENDIF` envelope 中
- 这段代码永远不会被执行（因为 OP_FALSE）
- 但数据永久存储在区块链上
- 享受 witness 的 75% 成本折扣

---

## 2.3 Output 的数据入口：两种根本模式

Output 结构是：

```
{
  value: <8 bytes>
  scriptPubKey: <locking script>
}
```

**关键理解：** 所有 Output 数据都存储在 `scriptPubKey` 字段中，但有两种根本不同的模式。

### 模式 A：数据直接存储（Data-in-ScriptPubKey）

数据直接暴露在 scriptPubKey 中，创建时永久可见，存储在 UTXO 集中。

**包含三种方式：**

1. **OP_RETURN** - scriptPubKey 的内容就是 `OP_RETURN <data>`（专门用于数据嵌入）
2. **Bare Multisig** - 将数据编码为"假公钥"放入多签脚本中
3. **scriptPubKey 本体** - 将数据伪装成可执行的脚本结构（如 `<push data> OP_DROP ...`）

**特点：**
- 数据创建时就永久可见
- 直接存储在 UTXO 集中
- 无成本折扣（4 WU/byte）

### 模式 B：承诺存储（Commitment-in-ScriptPubKey）

scriptPubKey 只包含承诺（hash 或 tweaked pubkey），实际数据在后续 Input 中揭示。

**包含两种方式：**

1. **P2SH/P2WSH** - Hash 承诺（详见 2.4 节）
2. **Taproot** - Merkle root 承诺（详见 2.4 节）

**特点：**
- Output 小巧（只有承诺）
- 数据在 Input witness/scriptSig 中揭示
- 享受 witness 折扣（1 WU/byte）

---

**本节（2.3）详细展开模式 A（数据直接存储），模式 B（承诺存储）在 2.4 节展开。**

### 2.3.1 OP_RETURN —— 最直接的数据入口（"明目张胆写数据"的方式）

脚本：

```
OP_RETURN <data>
```

**数据位置：直接在 Output 的 scriptPubKey 中**

特点：

- 不可花费（consensus）
- 节点无需验证任何逻辑
- 每个输出独立
- 数据由外部协议解释
- **数据创建时就永久可见**

限制（v30之前）：

- block policy 限制大小（80 bytes → 可调）
- 单次输出不能太大（避免 spam）
- 单个交易最多允许 1 个 OP_RETURN 输出

限制（v30之后）：

- block policy 限制大小提升至 100,000 bytes（实际受交易大小和区块 weight 约束）
- 允许多个 OP_RETURN 输出
- 通过 `-datacarriersize` 参数可配置

**OP_RETURN 大小限制的演变：**

- **2014年（Bitcoin Core 0.9）**：40 bytes
- **2015年（Bitcoin Core 0.11）**：80 bytes
- **2017年至今（v0.11-v29）**：默认 80 bytes（policy层可配置）
- **2025年10月（Bitcoin Core v30）**：100,000 bytes（实际受交易大小和区块 weight 约束）
  - 首次允许多个 OP_RETURN 输出
  - 通过 `-datacarriersize` 参数可配置
  - 这一变化在社区中引发争议，Bitcoin Knots 等实现选择保持 80 bytes 限制

实际上，consensus 层对 OP_RETURN 大小没有硬性限制，但：

- v30之前：mempool policy 默认拒绝超过 80 bytes 的 OP_RETURN
- v30之后：policy 层限制大幅放宽至 100,000 bytes，并允许多个 OP_RETURN 输出
- 矿工可以自行调整接受标准（通过 `-datacarriersize` 参数）

OP_RETURN 是：

- ✔ 最清晰的数据入口
- ✔ 也是最被 Core 开发者警惕的入口
- ✔ 同时是 Omni、Counterparty（后期）、Runes 的基础

### 2.3.1 OP_RETURN 示例 - USDT (Omni Layer)

**交易：** `21c3354ab4422faca43761826884bb4398871a2c404a2fff71be91c9502d3ba0`

[Mempool 链接](https://mempool.space/tx/21c3354ab4422faca43761826884bb4398871a2c404a2fff71be91c9502d3ba0)

这是一个典型的 USDT 转账交易（基于 Omni 协议）。

**Output #1 (OP_RETURN):**

```
scriptPubKey hex:
6a146f6d6e69000000000000001f000000003b9aca00

解析：
6a          = OP_RETURN
14          = PUSH 20 bytes
6f6d6e69    = "omni" (ASCII)
00000000    = Transaction version
0000001f    = Transaction type (Simple Send)
0000001f    = Property ID 31 (USDT)
000000003b9aca00 = 金额 (10.00000000 USDT)
```

**关键点：**

- OP_RETURN output 的 value 为 0
- 最多 80 字节数据（v30 之前）
- 协议解析器读取这些字节来理解转账意图
- 这是 Omni Layer 协议的标准格式

### 2.3.2 Bare Multisig —— 把数据塞成"假的公钥"

比如：

```
OP_1 <fake_pubkey> <fake_pubkey> <fake_pubkey> OP_3 OP_CHECKMULTISIG
```

一个公钥是 33 bytes
一个多签可以放 N 个公钥
这些公钥可以是任意字节
→ **Stamp 协议就是这样做的。**

**数据位置：直接在 Output 的 scriptPubKey 中（假公钥）**

这条路径极其强大，但也极其危险。

Consensus 允许，policy 早期允许，后来不鼓励。

**为什么 Stamps 选择这种方式？**

- 想让数据永久在 output（UTXO 集）
- 不想依赖 witness（早期 Stamps 设计时 witness 还未普及）
- 数据创建时就可见，不需要解锁

### 2.3.3 Fake Multisig 示例 - Stamp 协议

**交易：** `b1278acf50c27342753d01af9013a709509fdd920bc40ddcbec0738aaec2764c`

[Mempool 链接](https://mempool.space/tx/b1278acf50c27342753d01af9013a709509fdd920bc40ddcbec0738aaec2764c)

Stamp 协议使用"假公钥"在 UTXO 中嵌入数据。

**特点：**

- 创建多个小额 output（通常 330-546 sats）
- 每个 output 的 scriptPubKey 看起来像正常地址
- 但公钥实际包含编码的图像数据
- 数据永久存在于 UTXO 集中

**示例 output 地址：**

```
bc1qapfywj2x8qukztqpcgqlxqqqqqqqqqrxqqq9zhp84vqqpq5t8lyqfvkk5y
bc1q5m4ywq8lcgq0hlxylh7axqqqqqqqqqqqqqqqqqqqqqqqqqqqqqsskk5xx9
```

**注意：** 地址中大量的 "q" 字符——这些是编码后的数据。

Stamp 协议将图像数据编码为多个假公钥，每个公钥 33 bytes，通过多个 output 分散存储，确保数据永久存在于 UTXO 集中。

### 2.3.3 scriptPubKey 本体 —— "数据伪装成脚本"

任何脚本只要能正确执行，都可以存在。

因此可以构造：

```
<push data> OP_DROP <standard script>
```

或：

```
<push big data> OP_IF ... OP_ENDIF
```

这些被称为 "poison scripts" 或 "大数据脚本"。

**数据位置：直接在 Output 的 scriptPubKey 中**

理论上，你可以塞任何数据，只要脚本结果为 true。

但：

- ❌ mempool policy 会拒绝非标准脚本
- ❌ 会引起永久性不可花费 UTXO（污染 UTXO 集）

所以不是主流数据嵌入方式。

### 2.3.4 实战案例对比

**OP_RETURN 示例 - USDT (Omni Layer)：**

交易：`21c3354ab4422faca43761826884bb4398871a2c404a2fff71be91c9502d3ba0`

- 数据直接暴露在 scriptPubKey 中
- 80 字节限制（v30 之前）
- 协议解析器直接读取

**Bare Multisig 示例 - Stamp 协议：**

交易：`b1278acf50c27342753d01af9013a709509fdd920bc40ddcbec0738aaec2764c`

- 数据编码为假公钥
- 多个 output 分散存储
- 数据永久存在于 UTXO 集中

**对比总结：**

| 方式 | 数据位置 | 容量 | 成本 | Policy 接受度 |
|------|---------|------|------|-------------|
| OP_RETURN | Output scriptPubKey | 80 bytes (v29) / 100KB (v30+) | 4 WU/byte | ✔ 高 |
| Bare Multisig | Output scriptPubKey | 中（每公钥 33 bytes） | 4 WU/byte | △ 低（Core 0.17+ 不中继） |
| scriptPubKey 本体 | Output scriptPubKey | 理论上大 | 4 WU/byte | ❌ 低（非标准脚本） |

---

## 2.4 承诺-揭示模式：Output 锁定，Input 揭示

这是比特币数据嵌入中最重要但最容易被误解的模式。

**核心原理：**

- **Output 侧**：只存储一个承诺（hash 或 tweaked pubkey），数据不在这里
- **Input 侧**：在 witness 中揭示实际数据
- **数据实际位置**：在 Input 的 witness 中

这种模式的优势：

- 数据享受 witness 的 75% 成本折扣
- Output 保持小巧（只有承诺）
- 数据在解锁时才揭示（可以选择性 reveal）

### 2.4.1 P2SH / P2WSH —— Hash 承诺模式

**P2SH（Pay to Script Hash）：**

Output 侧（创建锁）：

```
scriptPubKey: OP_HASH160 <20-byte-hash> OP_EQUAL
```

这里只有一个 hash 承诺，**没有实际数据**。

Input 侧（解锁时）：

```
scriptSig: <redeemScript>   # legacy
```

或（SegWit 版本）：

```
witness: [
  <signature>,
  <pubkey>,
  <witnessScript>  ← 实际数据在这里！
]
```

**数据实际位置：在 Input 的 witness 中（witnessScript）**

**P2WSH（Pay to Witness Script Hash）：**

Output 侧（创建锁）：

```
scriptPubKey: OP_0 <32-byte-sha256(witnessScript)>
```

这里只有一个 hash 承诺，**没有实际数据**。

Input 侧（解锁时）：

```
witness: [
  <signature>,
  <pubkey>,
  <witnessScript>  ← 实际数据在这里！
]
```

**数据实际位置：在 Input 的 witness 中（witnessScript）**

**关键洞察：**

- P2WSH 把数据放在 **input 的 witness 里**，不是 output！
- Output 只是一个"hash 锁"
- 解锁时才在 witness 里 reveal 实际的 witnessScript
- 这就是为什么 P2WSH 可以承载大数据——因为数据在 witness 里，享受 75% 折扣

这两者都可以使 script 变成：

- 自定义数据结构
- 状态机
- 元数据承载
- 子协议载体
- Merkle leaf 容器

P2WSH 是真正意义上的"数据容器"，但数据在 witness 中。

### 2.4.2 Taproot（P2TR）—— Merkle 承诺模式

**Taproot 的 script path：**

Output 侧（创建锁）：

```
scriptPubKey: OP_1 <32-byte-tweaked-pubkey>
```

这里只有一个 tweaked 公钥，包含了 Merkle root 的承诺，**没有实际数据**。

Input 侧（script path 解锁时）：

```
witness: [
  <signature/data>,
  <script leaf>,       ← 实际脚本在这里！
  <control block>      ← Merkle proof 在这里！
]
```

**数据实际位置：在 Input 的 witness 中（script leaf + control block）**

**关键洞察：**

- Taproot 也把数据放在 **input 的 witness 里**！
- Output 只是一个"承诺"（tweaked pubkey）
- 解锁时在 witness 里 reveal script leaf 和 Merkle path

你可以构建：

- 巨大的 Merkle 树
- 每一个 leaf 储存结构化数据
- 用 control block 选择性 reveal
- 编码复杂协议状态

Ordinals、Atomicals、RGB、Taproot Assets 都依赖它。

**为什么 Ordinals 用 Taproot 而不用 OP_RETURN？**

- Taproot 的数据在 witness（75% 折扣）
- OP_RETURN 的数据在 output（无折扣）
- Taproot 可以承载更大的数据

**为什么 RGB 用 Taproot commitment？**

- Output 侧只放 hash 承诺（小）
- 数据在链下（不上链）
- 链上只存储最小承诺

**Taproot 是目前比特币上"最结构化的数据入口"，但数据在 witness 中。**

---

## 2.5 数据位置对比表

| 方式 | Output 的角色 | Input 的角色 | 数据实际在哪？ | 成本 |
|------|--------------|-------------|--------------|------|
| OP_RETURN | 直接包含数据 | N/A（不可花费） | **Output** | 4 WU/byte |
| Bare multisig | 假公钥=数据 | 不需要 reveal | **Output** | 4 WU/byte |
| P2WSH | Hash 承诺 | Witness reveal witnessScript | **Input witness** | 1 WU/byte（witness部分） |
| Taproot | Tweaked pubkey 承诺 | Witness reveal script leaf | **Input witness** | 1 WU/byte（witness部分） |

**关键区别：**

- **"数据在 output"** = 数据永久可见，创建时就公开
  - OP_RETURN
  - Bare multisig（Stamps）

- **"数据在 input"** = 数据在解锁时才 reveal
  - P2WSH witnessScript
  - Taproot script leaf
  - 都在 witness 里

---

## 2.6 哪些字段通常不作为自由数据空间？

以下字段通常不用于嵌入任意数据：

- **version** - 交易版本号（共识关键）
- **locktime** - 时间锁（共识字段）
- **value** - 金额（货币单位）
- **sequence** - 序列号（见下文特殊说明）

### 2.6.1 特殊案例：sequence 字段

**sequence 字段的设计用途：**

- 原始用途：允许交易在确认前被替换（已废弃）
- BIP 68：相对时间锁（CheckSequenceVerify）
- 用于控制交易的时间锁逻辑

**但历史上有过"滥用"案例：**

早期的 EPOBC（Enhanced Padded Order-Based Coloring）协议曾经利用 sequence 的 32-bit 空间编码彩色币元数据。

**示例：**

```
nSequence = 0xC0L0RFFF

           ↑  ↑
           |  └─ 颜色ID和金额
           └─ 协议标记位
```

**为什么 EPOBC 失败了？**

1. ❌ 每次转账都需要追溯到创世交易（效率低）
2. ❌ 没有外部索引器就无法验证状态
3. ❌ 与 BIP 68 时间锁功能冲突
4. ❌ 数据空间太小（仅 32 bits）

**教训：** sequence 不是设计用来存储任意数据的，但它证明了一个重要原则——**人们会尝试利用交易中的每一个字节**。

详见第 3 章关于彩色币的完整历史。

---

## 2.7 Consensus vs Policy：为什么某些方式被拒绝？

很多嵌入方式本身并不违反共识：

- Witness 允许任意 bytes
- OP_RETURN 的大小在 policy 层可调
- 多签的 pubkey technically 可以是任意 33 bytes
- Taproot leaf 可包含任意脚本结构

问题出在 **policy 层**：

Bitcoin Core 可以拒绝中继（relay）某些交易，即使它们在共识层合法。

例如：

- 超大 witness
- 非标准脚本
- 裸多签（bare multisig）
- 无意义的 OP_RETURN spam

因此某些协议能"上链"，但无法"进入公众 mempool"。

这为后续章节理解 Stamps、Counterparty、Ordinals 等协议争议奠定背景。

### Consensus vs Policy 的具体边界

**Consensus层允许，但Policy层（Bitcoin Core默认）拒绝的情况：**

1. **超大交易**：超过 400,000 weight units
2. **非标准脚本**：除非被 P2SH/P2WSH/P2TR 包装
3. **多个 OP_RETURN**：v30之前单笔交易超过 1 个 OP_RETURN 输出（v30+已允许多个）
4. **Dust outputs**：小于 546 satoshis（legacy）或 294 satoshis（SegWit）
5. **裸多签（Bare multisig）**：Bitcoin Core 0.17+ 不再中继

**Policy的目的是：**

- 防止 UTXO 集膨胀
- 避免区块空间滥用
- 保护节点资源

但矿工可以不遵守这些policy，直接打包符合consensus的交易。

这就是为什么某些"非标准"协议（如早期Stamps）仍能上链，但无法通过公共mempool传播。

---

## 2.8 数据入口能力对照表

| 入口方式 | 数据实际位置 | 共识允许 | policy 允许 | 容量 | Weight成本 | 引入时间 | 典型协议 |
|---------|------------|---------|------------|------|-----------|---------|---------|
| scriptSig | Input scriptSig | ✔ | △ | 中 | 4 WU/byte | 2009 | 彩色币 |
| witness（直接） | Input witness | ✔✔ | ✔✔ | 极大 | 1 WU/byte | 2017 (BIP 141) | Ordinals / Atomicals |
| OP_RETURN | **Output** | ✔ | ✔ | 80 bytes (v29) / 100KB (v30+) | 4 WU/byte | 2014 | Omni / Runes |
| multisig 伪 pubkey | **Output** | ✔ | △ | 中 | 4 WU/byte | 2009 | Stamp |
| P2WSH | **Input witness** | ✔✔ | ✔✔ | 极大 | 1 WU/byte | 2017 (BIP 141) | Counterparty (SW) |
| Taproot leaf | **Input witness** | ✔✔ | ✔✔ | 大且结构化 | 1 WU/byte | 2021 (BIP 341) | Atomicals / RGB |

**经济效率排名：**

1. ⭐⭐⭐⭐⭐ Witness（75%折扣）- 包括直接 witness、P2WSH、Taproot
2. ⭐⭐ OP_RETURN（全部base weight）
3. ⭐ Bare multisig（已不推荐）

**数据位置分类：**

- **数据在 Output**：OP_RETURN、Bare multisig
- **数据在 Input Witness**：直接 witness、P2WSH、Taproot

简而言之：

- **Witness / P2WSH / Taproot** 是现代三大入口（数据在 witness，享受折扣）
- **OP_RETURN** 是直接 output 入口（无折扣但简单）
- **scriptSig、裸多签**属于"历史遗迹"

---

## 2.9 数据嵌入的四个时代

理解比特币数据嵌入史，需要把握四个技术时代的转折点：

### **Legacy Era (2009-2017)：原始探索**

```
scriptSig → 彩色币的花费路径编码
OP_RETURN → Omni / Counterparty 的指令层
Bare multisig → 早期 Stamps / Counterparty 的数据容器
```

特点：

- 数据与交易逻辑混杂
- 容易产生 UTXO 污染
- 缺乏结构化设计
- 数据主要在 output（除了 scriptSig）

---

### **SegWit Era (2017-2021)：Witness 过渡期**

```
witness → Counterparty 协议升级
P2WSH → 结构化脚本容器（数据在 witness）
witness discount → 大数据嵌入的经济可行性
```

特点：

- 数据与 TXID 解耦
- 75% 成本折扣
- 数据从 output 迁移到 witness
- 为下一代协议铺路
- 这是 witness 技术的过渡和探索期

---

### **Taproot Era (2021-2025)：百花齐放**
```
witness + Taproot script-path + 序数理论 → Ordinals 铭文（数据在 witness）
BRC-20 → 基于 Ordinals 的 JSON 代币标准
witness + Taproot script-path + colored coins 机制 → Atomicals (ARC-20)
Taproot commitments (tapret/opret) → RGB 链上承诺 + 链下状态
Runes → OP_RETURN + UTXO 模型的代币协议（不使用 Taproot）
```

特点：

- Merkle 树结构化数据
- 选择性 reveal（Taproot script-path）
- 链上承诺，链下验证（RGB）
- Witness 数据享受 75% 费用折扣
- **协议百花齐放**：Ordinals、BRC-20、Atomicals、RGB、Runes 等协议相继涌现
- **技术路线分化**：既有基于 Taproot + witness 的创新（Ordinals/Atomicals），也有回归 OP_RETURN 的简化方案（Runes）
- 这是比特币数据嵌入最活跃的时期

---

### **OP_RETURN v30 Era (2025-now)：政策转折与讨论**
```
OP_RETURN v30 → 100KB 限制 + 多个输出（2025年10月）
Bitcoin Core vs Bitcoin Knots → 实现多样性
社区分裂 → 关于 Bitcoin 本质的哲学辩论
```

特点：

- OP_RETURN 从 80 bytes 放宽至 100KB
- 首次允许多个 OP_RETURN 输出
- 核心开发者出现倾向性意见分歧
- 社区讨论激烈，观点对立
- 通过实现多样性（Bitcoin Core vs Bitcoin Knots）而非强制统一来应对分歧

**说明：** v30 的发布标志着 Bitcoin 数据嵌入政策史上的重要转折点。这一变化不仅改变了技术限制，更引发了关于 Bitcoin 本质的哲学辩论——Bitcoin 应该是纯粹的"电子现金"，还是可以承载更多数据和应用？目前讨论仍在进行中，后续发展有待观察。

每个时代都是对"如何在比特币塞数据"的回答升级。

---

## 2.10 总结：所有协议都只是"入口的不同组合"

第 2 章到这里，我们已经准备好阅读整个领域。

**核心洞察：**

所有非交易数据在比特币中的存在，都源自 input 或 output。
但更重要的是，需要理解**数据实际存储的位置**：

- **数据在 Output**：创建时永久可见，无成本折扣
- **数据在 Input Witness**：解锁时揭示，享受 75% 折扣

你会看到：

- **Omni** → OP_RETURN（数据在 output）
- **Counterparty** → OP_RETURN + multisig（数据在 output）
- **Stamp** → multisig（数据在 output）
- **Ordinals** → Witness（数据在 witness）
- **Atomicals** → Witness + Taproot Merkle（数据在 witness）
- **Runes** → OP_RETURN 现代化（数据在 output）
- **RGB** → Taproot commitment + off-chain state（承诺在 output，数据在链下）

每一种都是：

**input vs output 的不同排列组合** + **数据实际位置的选择**。

理解本章，你能一眼看穿这些协议。

本书的后续所有章节，将沿着以下顺序展开：

- **Output 的数据系谱：**
  彩色币 → Omni → Counterparty → Runes
- **Input / Witness 的数据系谱：**
  Stamps 裸多签 → P2WSH → Ordinals → Atomicals
- **Taproot Merkle 的结构化路线**
- **以及最终的 RGB（off-chain state machine）**

这些不是孤立事件，而是**比特币数据嵌入的统一历史**。

---

**理解本章后，你在阅读后续章节时会发现：**

- **Ch3-6（Output系谱）**：彩色币、Omni、Counterparty、Stamps 如何滥用/正确使用 scriptPubKey 和 OP_RETURN（数据在 output）
- **Ch7-11（Witness系谱）**：P2WSH、Ordinals、Atomicals 如何将 witness 推向极限（数据在 witness）
- **Ch12-13（未来方向）**：RGB 如何彻底跳出"数据上链"的思维

每一章都会回到本章建立的"数据入口地图"。

每一个协议的本质，都是这些入口的**不同排列组合** + **不同解释层** + **数据位置的选择**。

如果你看懂了 input 与 output 的所有可能性，以及数据实际存储的位置，你就看懂了比特币非交易数据的全部历史。

---

**下一章预告：** 

从最早的彩色币开始，我们将看到人们如何一步步发现和利用这些数据入口，最终演化出今天的 Ordinals、Stamps 等协议。

**实验部分：** 详见 Chapter 2 Part 2，我们将通过测试网实战展示从裸脚本到 Commit-Reveal 模式的完整演进。

