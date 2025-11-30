# Chapter 2

## 比特币数据入口全图：Input vs Output 的完整结构

### The Complete Map of Where Data Can and Cannot Enter Bitcoin

上一章说明了一个核心事实：

比特币只保留两类字节：输入数据（用于解锁）与输出数据（用于上锁）。
所有非交易数据的写入，都只能沿着这两个方向展开。

但这句话本身仍然过于抽象。
本章将逐层拆开实际的比特币交易结构，明确：
哪些字段可以承载字节、承载多少、有什么规则、哪些是共识 vs 策略、哪些可以滥用、哪些不可滥用。

本章的内容可以直接作为一个"数据嵌入完整地图"使用，帮助理解所有协议的底层技术基础。

---

## 2.1 比特币交易的字节结构总览

比特币的交易（Transaction）由四大块组成：

```
Version
Inputs[]
Outputs[]
Locktime
```

其中**只有 Inputs 与 Outputs 包含"可插入字节"的空间**。

下面我们展开每一个部分。

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

---

## 2.3 Output 的数据入口

Output 结构是：

```
{
  value: <8 bytes>
  scriptPubKey: <locking script>
}
```

其中可装数据的部分就是 **scriptPubKey**。

scriptPubKey 可以是：

- P2PKH
- P2SH
- P2WPKH
- P2WSH
- P2TR
- OP_RETURN

每一种代表了不同级别的数据承载能力。

我们逐一分析。

### 2.3.1 OP_RETURN —— 最直接的数据入口（"明目张胆写数据"的方式）

脚本：

```
OP_RETURN <data>
```

特点：

- 不可花费（consensus）
- 节点无需验证任何逻辑
- 每个输出独立
- 数据由外部协议解释

限制：

- block policy 限制大小（80 bytes → 可调）
- 单次输出不能太大（避免 spam）

**OP_RETURN 大小限制的演变：**

- **2014年（Bitcoin Core 0.9）**：40 bytes
- **2015年（Bitcoin Core 0.11）**：80 bytes
- **2017年至今**：默认 80 bytes（policy层可配置更大）

实际上，consensus 层对 OP_RETURN 大小没有硬性限制，但：

- mempool policy 默认拒绝超过 80 bytes 的 OP_RETURN
- 矿工可以自行调整接受标准
- 单个交易最多允许 1 个 OP_RETURN 输出（policy层）

OP_RETURN 是：

- ✔ 最清晰的数据入口
- ✔ 也是最被 Core 开发者警惕的入口
- ✔ 同时是 Omni、Counterparty（后期）、Runes 的基础

### 2.3.2 scriptPubKey 本体 —— "数据伪装成脚本"

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

理论上，你可以塞任何数据，只要脚本结果为 true。

但：

- ❌ mempool policy 会拒绝非标准脚本
- ❌ 会引起永久性不可花费 UTXO（污染 UTXO 集）

所以不是主流数据嵌入方式。

### 2.3.3 多签（multisig）—— 把数据塞成"假的公钥"

比如：

```
OP_1 <fake_pubkey> <fake_pubkey> <fake_pubkey> OP_3 OP_CHECKMULTISIG
```

一个公钥是 33 bytes
一个多签可以放 N 个公钥
这些公钥可以是任意字节
→ **Stamp 协议就是这样做的。**

这条路径极其强大，但也极其危险。

Consensus 允许，policy 早期允许，后来不鼓励。

### 2.3.4 P2SH / P2WSH —— redeemScript / witnessScript 当作"结构化容器"

**P2SH：**

```
OP_HASH160 <20-byte-hash> OP_EQUAL
```

后台：redeemScript（可以很大）

**P2WSH：**

```
OP_0 <32-byte-sha256(script)>
```

后台：witnessScript（可以更大）

这两者都可以使 script 变成：

- 自定义数据结构
- 状态机
- 元数据承载
- 子协议载体
- Merkle leaf 容器

P2WSH 是真正意义上的"数据容器"。

它是 Ordinals 之前最严肃的数据入口。

### 2.3.5 Taproot（P2TR）—— Merkle 化的数据结构时代

Taproot 的 script path：

```
control block + script leaf
```

leaf 脚本：

```
<push data> OP_DROP <whatever>
```

你可以构建：

- 巨大的 Merkle 树
- 每一个 leaf 储存结构化数据
- 用 control block 选择性 reveal
- 编码复杂协议状态

Ordinals、Atomicals、RGB、Taproot Assets 都依赖它。

**Taproot 是目前比特币上"最结构化的数据入口"。**

---

## 2.4 哪些字段不能写数据？

以下字段不适合作为数据输入：

- **version**（共识-critical）
- **locktime**（共识字段）
- **sequence**（状态控制字段，但也是一种特殊的"时间数据"编码）
  - BIP 68: 相对时间锁（CSV - CheckSequenceVerify）
  - 虽然不能塞任意数据，但可以编码时间状态
  - 这属于"结构化的数据编码"而非自由字节空间
- **value**（货币单位）

它们不是自由字节空间，不能滥用。

---

## 2.5 Consensus vs Policy：为什么某些方式被拒绝？

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
3. **多个 OP_RETURN**：单笔交易超过 1 个 OP_RETURN 输出
4. **Dust outputs**：小于 546 satoshis（legacy）或 294 satoshis（SegWit）
5. **裸多签（Bare multisig）**：Bitcoin Core 0.17+ 不再中继

**Policy的目的是：**

- 防止 UTXO 集膨胀
- 避免区块空间滥用
- 保护节点资源

但矿工可以不遵守这些policy，直接打包符合consensus的交易。

这就是为什么某些"非标准"协议（如早期Stamps）仍能上链，但无法通过公共mempool传播。

---

## 2.6 数据入口能力对照表

| 入口方式 | 共识允许 | policy 允许 | 容量 | Weight成本 | 引入时间 | 典型协议 |
|---------|---------|------------|------|-----------|---------|---------|
| scriptSig | ✔ | △ | 中 | 4 WU/byte | 2009 | 彩色币 |
| witness | ✔✔ | ✔✔ | 极大 | 1 WU/byte | 2017 (BIP 141) | Ordinals / Atomicals |
| OP_RETURN | ✔ | ✔ | 80 bytes | 4 WU/byte | 2014 | Omni / Runes |
| multisig 伪 pubkey | ✔ | △ | 中 | 4 WU/byte | 2009 | Stamp |
| P2WSH script | ✔✔ | ✔✔ | 极大 | 混合 | 2017 (BIP 141) | Counterparty (SW) |
| Taproot leaf | ✔✔ | ✔✔ | 大且结构化 | 混合 | 2021 (BIP 341) | Atomicals / RGB |

**经济效率排名：**

1. ⭐⭐⭐⭐⭐ Witness（75%折扣）
2. ⭐⭐⭐ Taproot leaf（部分witness，部分base）
3. ⭐⭐ OP_RETURN（全部base weight）
4. ⭐ Bare multisig（已不推荐）

简而言之：

- **OP_RETURN / Witness / Taproot** 是现代三大入口
- **scriptSig、裸多签**属于"历史遗迹"

---

## 2.6.5 数据嵌入的三个时代

理解比特币数据嵌入史，需要把握三个技术时代的转折点：

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

---

### **SegWit Era (2017-2021)：Witness 的解放**

```
witness → Counterparty 协议升级
P2WSH → 结构化脚本容器
witness discount → 大数据嵌入的经济可行性
```

特点：

- 数据与 TXID 解耦
- 75% 成本折扣
- 为下一代协议铺路

---

### **Taproot Era (2021-now)：Merkle 化与结构化**

```
Taproot leaves → Atomicals / RGB 的数据分层
witness + Taproot → Ordinals 的图片铭文
Taproot commitments → 链上承诺 + 链下状态
```

特点：

- Merkle 树结构
- 选择性 reveal
- 链上承诺，链下验证

每个时代都是对"如何在比特币塞数据"的回答升级。

---

## 2.7 总结：所有协议都只是"入口的不同组合"

第 2 章到这里，我们已经准备好阅读整个领域。

你会看到：

- **Omni** → OP_RETURN
- **Counterparty** → OP_RETURN + multisig
- **Stamp** → multisig
- **Ordinals** → Witness
- **Atomicals** → Witness + Taproot Merkle
- **Runes** → OP_RETURN 现代化
- **RGB** → Taproot commitment + off-chain state

每一种都是：

**input vs output 的不同排列组合。**

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

- **Ch3-6（Output系谱）**：彩色币、Omni、Counterparty、Stamps 如何滥用/正确使用 scriptPubKey 和 OP_RETURN
- **Ch7-11（Witness系谱）**：P2WSH、Ordinals、Atomicals 如何将 witness 推向极限
- **Ch12-13（未来方向）**：RGB 如何彻底跳出"数据上链"的思维

每一章都会回到本章建立的"数据入口地图"。

每一个协议的本质，都是这些入口的**不同排列组合** + **不同解释层**。

如果你看懂了 input 与 output 的所有可能性，你就看懂了比特币非交易数据的全部历史。
