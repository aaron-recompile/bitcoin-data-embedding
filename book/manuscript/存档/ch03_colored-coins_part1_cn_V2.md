# Chapter 3

## 彩色币（Colored Coins）的工程复现

### 第一代比特币资产协议：从"零数据"到 OP_RETURN 的演进

彩色币（Colored Coins）是比特币历史上第一个链上资产协议。  
它开创了一个革命性的理念：

**UTXO 不仅仅承载 BTC，还可以承载"颜色"——即链外解释的资产属性。**

与后来的 Omni、Counterparty、Stamps、Ordinals、Atomicals 完全不同，  
彩色币最令人惊讶的两点是：

1. **早期实现几乎不写链上数据**（EPOBC 使用 nSequence 字段，零开销）
2. **后期实现使用 OP_RETURN**（Open Assets Protocol，2013年末）

本章将从工程角度深入理解这一结构：

- 彩色币的两大技术路线（EPOBC vs Open Assets）
- 核心技术原理与设计权衡
- 为什么它们能工作
- 为什么最终失败
- 对后续协议的深远影响

**💡 实践提示：**  
本章专注于原理和概念。完整的代码实现、测试和动手实践请参考 **Chapter 3 Part 2: 彩色币协议动手实践**。

---

## 3.1 历史背景：2012-2015 年的比特币"资产觉醒"

### 3.1.1 时间线与关键人物

彩色币概念最早由 eToro 创始人 Yoni Assia 于 2012年3月27日在博客文章 "bitcoin 2.X (aka Colored Bitcoin) - initial specs" 中提出。2012年12月4日，以色列比特币协会主席 Meni Rosenfeld 发表了更系统的白皮书 "Overview of Colored Coins"。

2013年，Yoni Assia、Meni Rosenfeld、Vitalik Buterin（后来的以太坊创始人）、Lior Hakim、Amos Meiri、Alex Mizrahi 和 Rotem Lev 共同发表了第二篇白皮书 "Colored Coins — BitcoinX"，探讨了彩色币的应用潜力。

技术实现方面：

- 2013年末：Flavien Charlon（后来成为 Coinprism CEO）提出 Open Assets Protocol，这是第一个工作的彩色币协议
- 2014年5月13日：Coinprism 钱包正式发布
- 2014年7月3日：ChromaWay 发布 Enhanced Padded-Order-Based Coloring (EPOBC) 协议
- 2015年6月：Colu 开源了基于 OP_RETURN 的新实现，支持 torrent 存储元数据

### 3.1.2 为什么需要彩色币？

在 2012-2013 年，比特币社区面临一个困境：

- 人们想在比特币上发行资产（股票、债券、商品凭证）
- 但当时没有标准化的数据嵌入方案
- OP_RETURN 直到 2014年才正式引入（Bitcoin Core 0.9）
- 社区普遍认为"比特币不是用来存储任意数据的"

在这样的环境下，彩色币提出了一个巧妙的思路：

**不用写太多数据，甚至不用写数据。**  
**只依赖 UTXO 的可追踪性和交易结构本身。**

---

## 3.2 彩色币核心思想：UTXO 携带"颜色"

彩色币协议允许在比特币交易中存储少量元数据，用于表示资产操作指令。核心理念是：

每个 UTXO 除了 BTC 价值，还可以带有一层"颜色（asset metadata）"。

示例：

```
UTXO_1 = 0.0001 BTC + "100 units of Asset A"
UTXO_2 = 0.0001 BTC + "50 shares of Company X"
UTXO_3 = 0.0001 BTC + "1 ticket to concert Y"
```

**关键洞察：**

颜色不是链上共识字段，而是链外客户端的解释层。

- 节点不知道颜色
- 矿工不关心颜色
- 验证规则不验证颜色

颜色仅存在于：**协议解释器（color-aware wallet/parser）**

这是一个**"off-chain 状态机"**。

---

## 3.3 两大技术路线

彩色币有两种主要实现方式，技术差异巨大：

### 路线 1：EPOBC（Enhanced Padded-Order-Based Coloring）

EPOBC 使用交易第一个输入的 nSequence 字段来标记彩色币交易类型。

**核心特点：**

- 零链上开销（nSequence 本来就存在且未被使用）
- 使用 nSequence 低6位编码标签
  - `110011` (0x33) = 转账交易
  - `100101` (0x25) = 发行交易
- 真正"着色" satoshis（颜色与具体聪绑定）
- EPOBC 是第一个支持 SPV 轻客户端的彩色币协议

### 路线 2：Open Assets Protocol

Open Assets 使用 OP_RETURN 作为 "marker output" 来存储资产数量信息。

**核心特点：**

- 使用 order-based coloring 算法映射资产
- Asset ID = RIPEMD160(SHA256(发行地址的 scriptPubKey))
- 需要专门的 marker output（OP_RETURN）
- 更灵活的元数据支持

---

## 3.4 EPOBC：零开销的大胆尝试

### 3.4.1 核心思想：不写数据的彩色币

EPOBC（Enhanced Padded-Order-Based Coloring）的设计哲学极其激进：

**能不能在完全不增加链上数据的前提下，实现资产协议？**

答案是：技术上可以，但实践中不行。

### 3.4.2 nSequence 字段的巧妙利用

每个比特币交易输入都有一个 4 字节的 `nSequence` 字段，原本用于交易替换机制。EPOBC 的创新在于：

**用 nSequence 的低 6 位来标记交易类型**

```
第一个输入的 nSequence & 0x3F:
  0x25 (37) → Genesis 交易（发行资产）
  0x33 (51) → Transfer 交易（转移资产）
  其他     → 普通比特币交易
```

**Genesis 交易逻辑：**
- 第一个输出即为"彩色币"
- 资产数量 = 输出的 satoshi 数量
- Color ID = "txid:vout"（简单但有效）

**Transfer 交易逻辑：**
- 收集所有彩色输入的总数量
- 按输出顺序依次分配
- 每个输出获得的数量 = min(输出金额, 剩余数量)

### 3.4.3 零开销的代价

**优势：**
- 真正的零字节开销
- 不污染 UTXO 集
- 交易看起来与普通比特币交易无异

**致命缺陷：**

1. **数量耦合**：资产数量 = satoshi 数量
   - 发行 100 万单位 → 需要 0.01 BTC
   - dust limit（546 sats）放大成本

2. **隐式标记**：完全依赖链外解释
   - 钱包不兼容会破坏颜色
   - 无法阻止误操作

3. **顺序依赖**：order-based coloring 太脆弱
   - UTXO 合并会混淆颜色
   - 输出重排序会导致错误分配

4. **缺乏元数据**：无法附加资产信息

EPOBC 证明了一个重要结论：**完全隐式的协议在开放网络中不可靠。**

---

## 3.5 Open Assets：OP_RETURN 的初步探索

### 3.5.1 显式标记的诞生

Open Assets Protocol 代表了彩色币的第二阶段进化：

**核心思想：既然隐式标记不可靠，那就显式地在链上写数据。**

2013 年末，Bitcoin Core 0.9.0 引入了 `OP_RETURN`，允许在交易中嵌入最多 40 字节的任意数据（后来扩展到 80 字节）。Open Assets 立即采用了这个新特性。

### 3.5.2 Marker Output：数据的容器

Open Assets 创建一个特殊的 "marker output"：

```
OP_RETURN 数据结构：
  0x6a           ← OP_RETURN opcode
  <length>       ← 数据长度
    0x4f 0x41    ← "OA" 魔数（协议标识）
    0x01 0x00    ← Version 1
    <count>      ← 资产数量个数
    <qty_list>   ← LEB128 编码的数量列表
    <metadata>   ← 可选元数据

例：[300, 700] 两个输出
  6a 0a 4f 41 01 00 02 ac 02 bc 05 00
```

**LEB128 编码**：可变长度整数编码，节省空间
- 小数字用 1 字节
- 大数字自动扩展
- 每字节用 7 位存数据，1 位表示是否继续

### 3.5.3 资产数量的解耦

这是 Open Assets 相对 EPOBC 的关键突破：

**资产数量与 BTC 金额分离**

```
EPOBC:
  1000 个资产单位 = 1000 satoshis（强制耦合）

Open Assets:
  1000 个资产单位 + 546 satoshis（最小 dust）
  数量信息存储在 marker output 中
```

**优势：**
- 发行成本大幅降低
- 可以表示任意大的数量
- 支持元数据（名称、符号等）

### 3.5.4 Asset ID：基于哈希的唯一标识

Open Assets 用加密哈希生成 Asset ID：

```
Asset ID = RIPEMD160(SHA256(发行地址的 scriptPubKey))
```

**为什么这样设计？**
- **唯一性**：SHA256 几乎不可能碰撞
- **发行者控制**：只有拥有私钥的人才能创建特定 Asset ID
- **可验证性**：任何人都可以验证 Asset ID 的来源
- **固定长度**：20 字节，与比特币地址一致

### 3.5.5 Order-Based Coloring 的延续

Open Assets 仍然使用 order-based coloring，但有所改进：

```
交易结构：
  Input 0:  彩色 UTXO (1000 units)
  Output 0: Marker (quantities: [300, 700])
  Output 1: 接收地址 A → 300 units
  Output 2: 接收地址 B → 700 units

映射规则：
  marker 后面的输出按顺序获得对应数量
```

**仍然存在的问题：**
- 依然依赖输出顺序
- 不能混合不同颜色的资产
- 钱包需要特殊支持

但相比 EPOBC，Open Assets 至少有了**明确的链上标记**，大大提高了可靠性。

---

## 3.6 彩色币的致命缺陷

### 3.6.1 技术缺陷

**1. 钱包行为不可控**

普通钱包的正常操作会立即破坏颜色：
- 自动合并多个输入（UTXO consolidation）
- 自动调整输出顺序以优化手续费
- 添加找零地址
- 使用 RBF（Replace-By-Fee）替换交易

**问题场景：**

用户持有混合 UTXO：
- UTXO_1: 包含 100 个 ColorA
- UTXO_2: 普通 BTC

钱包自动合并这两个 UTXO 创建交易时：
- 100 个 ColorA 应该分配给哪个输出？
- 按金额分配？按顺序分配？
- 普通 BTC 会"稀释"颜色吗？
- 不同钱包的实现不同 → 颜色丢失

**2. 缺乏原子性保证**

资产交换场景（Alice 想用 50 ColorA 换 Bob 的 0.01 BTC）：
- 如何保证同时交换？
- 没有脚本级别的原子性
- 必须依赖链外协调
- 容易被一方欺诈

彩色币无法在协议层面提供原子交换，只能依赖信任或第三方托管。

**3. Dust Limit 的影响**

Bitcoin Core 引入 dust limit 来防止小额输出污染 UTXO 集：

```
历史演变：
  早期（~2013-2014）: 5460 satoshis
  后来优化（~2015+）: 546 satoshis (P2PKH)
  SegWit 时代：      294 satoshis (P2WPKH)
```

**对 EPOBC 的影响：**

由于 EPOBC 的资产数量与 satoshi 金额直接耦合：

```
理论上：1 个资产单位 = 1 satoshi
实际上：1 个资产单位 ≥ 546 satoshis

问题：
- 发行 100 万单位 → 需要 0.00546 BTC
- 成本放大 546 倍
- 小额资产经济上不可行
```

**对 Open Assets 的影响较小：**

Open Assets 的数量与金额解耦，每个输出只需最小 dust（546 sats），无论承载多少资产单位。这是 Open Assets 相对 EPOBC 的关键优势之一。

### 3.6.4 典型攻击场景：颜色为何容易丢失

让我们通过具体场景理解彩色币的脆弱性：

**场景 1：不兼容钱包的破坏（EPOBC）**

```
用户 Alice 持有 1000 个彩色币（在一个 UTXO 中）

Alice 用普通比特币钱包发起转账：
  → 钱包自动选择 UTXO，包括那个彩色 UTXO
  → 钱包不知道需要保持 nSequence 标记
  → 创建找零时 nSequence 被重置为 0xFFFFFFFF
  
结果：
  ✗ 彩色币标记丢失
  ✗ 1000 个资产单位永久变回普通 BTC
  ✗ 用户甚至不知道发生了什么
```

**场景 2：UTXO 合并混淆（两种协议都有）**

```
Alice 有两个彩色 UTXO：
  UTXO_A: 500 units of Asset_X
  UTXO_B: 300 units of Asset_Y（不同资产！）

普通钱包合并 UTXO 创建单笔转账：
  Input: [UTXO_A, UTXO_B]
  Output: 一个找零地址
  
Order-Based Coloring 的困境：
  - 800 units of Asset_X？
  - 500 units of Asset_X + 300 units of Asset_Y？
  - 协议无法处理混合资产
  
结果：颜色混淆，资产丢失或错误分配
```

**场景 3：输出重排序攻击（Open Assets）**

```
正常转账交易：
  Marker: [300, 700]
  Output 1: Alice 的地址 → 应得 300
  Output 2: Bob 的地址 → 应得 700

如果恶意节点重排序输出：
  Marker: [300, 700]（位置不变）
  Output 1: Bob 的地址 → 获得 300
  Output 2: Alice 的地址 → 获得 700
  
结果：
  ✗ 资产被重新分配
  ✗ 协议层面无法防御
  ✗ 必须信任交易广播的诚实性
```

**根本原因分析：**

1. **缺乏密码学保护**：没有签名直接绑定资产分配
2. **链外依赖**：必须信任所有节点的链外解释一致
3. **被动验证**：无法主动拒绝错误交易
4. **顺序脆弱性**：order-based coloring 过于依赖外部约束

这些攻击场景清晰地展示了为什么彩色币无法成为生产级协议：**在开放、不可信的网络中，隐式和半隐式协议都太脆弱了。**

**4. 缺乏元数据表达能力**

早期 EPOBC 协议只能表达：
- 资产数量
- 颜色 ID

无法表达：
- 资产名称和符号
- 发行者信息
- 可分割性规则
- 过期时间
- 转让限制
- 任何智能合约逻辑

Open Assets 虽然支持元数据，但由于 OP_RETURN 大小限制（40-80 bytes），也无法承载复杂信息。

### 3.6.2 生态与市场问题

**用户体验差：**
- 必须使用专门的彩色币钱包
- 不同协议之间不兼容
- 误操作会永久丢失资产
- 普通用户难以理解

**市场采用度低：**
- 缺乏杀手级应用
- 交易所不支持
- 流动性差
- 没有形成网络效应

**开发者生态分裂：**
- 多个协议竞争（EPOBC、Open Assets、Colu、CoinSpark）
- 缺乏统一标准
- 工具和文档不完善
- 社区共识难以达成

---

## 3.7 为什么彩色币最终失败？

总结其失败的根本原因：

### (1) 缺乏强制执行机制

比特币共识层完全不关心"颜色"：
- ✅ 验证签名
- ✅ 验证输入金额 ≥ 输出金额
- ✅ 验证没有双花
- ❌ **不验证颜色是否正确传播**
- ❌ **不验证资产数量是否守恒**

彩色币的规则完全在链外，无法被网络强制执行。任何破坏颜色的交易在比特币层面都是合法的。

### (2) 钱包生态不兼容

彩色币需要专门的钱包：
- Coinprism (Open Assets)
- ChromaWallet (EPOBC)
- Colu (Colu Protocol)
- CoinSpark (CoinSpark Protocol)

问题：
- 用户必须使用特定钱包
- 不同彩色币协议之间不兼容
- **一旦发送到普通钱包 → 颜色永久丢失**
- 生态分裂，无法形成网络效应

### (3) 缺乏链上承诺（commitment）

彩色币的数据几乎全在链外：
- EPOBC：只有 nSequence 标记（6 bits）
- Open Assets：有 OP_RETURN 但不够完善

缺乏的关键要素：
- ❌ 没有加密学证明
- ❌ 没有 Merkle 承诺
- ❌ 状态无法独立验证
- ❌ 必须信任链外索引器
### (4) 经济成本问题（EPOBC）

由于资产数量与 satoshi 耦合，dust limit 大幅提高了成本：

```
发行 100 万个单位：

理论成本：1,000,000 sats = 0.01 BTC
实际成本（546 dust limit）：546,000,000 sats = 5.46 BTC

如果 BTC = $40,000
实际美元成本 = $218,400

放大倍数：546x
```

这让 EPOBC 在经济上完全不可行。Open Assets 虽然解耦了数量，但仍需为每个 UTXO 支付 dust。

---

## 3.8 彩色币的历史贡献

尽管彩色币失败了，但它的思想在后续协议中不断被继承和发展：

### ① 提出了"链外状态机"模型

**彩色币的思路：**
- 状态保存在链外解释中
- 链上只有最小标记（或无标记）
- 客户端解析构建状态

**后续演进：**
- **RGB Protocol**：完善的客户端验证 + Taproot 承诺
- **Taproot Assets**：基于 Taproot 的链外状态
- 核心理念相同，但加入了密码学保护

### ② 证明了资产可以依附在 UTXO 之上

彩色币的核心洞察：**资产是对 UTXO 的额外解释层**

这个抽象被所有后续协议继承：
- **Omni**：UTXO + OP_RETURN 数据
- **Counterparty**：UTXO + 编码数据
- **Ordinals**：UTXO + 序数理论
- **RGB**：UTXO + 客户端状态 + 链上承诺

没有彩色币的探索，就不会有这些协议。
```

### ③ 启发了 OP_RETURN 的引入

彩色币开发者的需求直接推动了 OP_RETURN 的标准化：

```
历史时间线：
  2012 → 彩色币提出，缺乏标准数据嵌入方式
  2013 → OP_RETURN 讨论，彩色币是主要推动力
  2014 → Bitcoin Core 0.9 引入 OP_RETURN (40 bytes)
  2016 → 扩展到 80 bytes
  2025 → 扩展到 100,000 bytes (v30)
```

没有彩色币对数据嵌入的探索，OP_RETURN 可能不会这么快被引入和标准化。

### ④ 确立了前提：复杂协议需要明确的数据结构

**彩色币的教训：**
- 隐式状态机 → 不可靠
- 零数据协议 → 不适合复杂应用
- 纯链外解释 → 需要链上承诺
- 忽视钱包兼容性 → 生态无法形成

**后续协议的改进：**
- **Omni**：明确的 OP_RETURN 指令集
- **Counterparty**：结构化 payload
- **RGB**：密码学承诺 + 完整的链外状态
- **Atomicals**：Taproot commitment + witness 数据

每一个后续协议都在彩色币的基础上解决了其暴露的问题。

---

---

## 3.9 教训与启示：为什么需要 Omni

彩色币的失败不是终点，而是起点。它的探索为后续协议指明了方向。

### 3.9.1 彩色币证明了什么

**可行性：**
- ✅ 比特币 UTXO 模型**可以**承载资产语义
- ✅ OP_RETURN **可以**成为数据容器
- ✅ 链外状态机**可以**工作

**不可行性：**
- ❌ 完全隐式的协议（EPOBC）太脆弱
- ❌ Order-based coloring 缺乏鲁棒性
- ❌ 没有明确规范的协议无法形成生态

### 3.9.2 后续协议需要解决的问题

**1. 明确的链上标记**

```
彩色币的教训：
  EPOBC：nSequence 标记太隐式
  Open Assets：OP_RETURN 标记是对的方向，但不够完善

后续改进：
  → 需要更强的协议标识符
  → 需要版本控制
  → 需要完整的数据格式规范
```

**2. 独立的状态验证**

```
彩色币的教训：
  完全依赖链外共识
  不同客户端可能有不同解释

后续改进：
  → 明确的状态转换规则
  → 可验证的状态历史
  → 确定性的验证算法
```

**3. 钱包兼容性**

```
彩色币的教训：
  普通钱包会破坏颜色
  
后续改进：
  → 更鲁棒的编码
  → 向后兼容的设计
  → 即使不识别协议也不会破坏数据
```

**4. 经济可行性**

```
彩色币的教训：
  EPOBC：数量与 BTC 耦合，成本高
  Open Assets：解耦了，但缺乏激励

后续改进：
  → 资产数量完全独立于 BTC 金额
  → 合理的链上成本模型
  → 可持续的经济设计
```

### 3.9.3 Omni 的诞生背景

2013 年末，就在 Open Assets 发布的同时，另一个团队在开发更雄心勃勃的项目：**Mastercoin**（后改名 Omni Layer）。

**Omni 的设计思路：**

1. **完整的数据协议**
   - 不只是资产数量，而是完整的交易类型系统
   - 支持多种操作：发行、转账、交易、DEx

2. **明确的 OP_RETURN 规范**
   - 固定的协议标识符
   - 结构化的数据编码
   - 版本化的消息格式

3. **成熟的状态机**
   - 确定性的共识规则
   - 完整的验证逻辑
   - 可审计的状态历史

4. **生产级应用**
   - 2014 年，USDT 选择在 Omni 上发行
   - 至今仍有数十亿美元的 USDT 在 Omni 上运行

**从彩色币到 Omni：**

```
彩色币：概念验证（Proof of Concept）
  → "能不能在比特币上发资产？"
  → 答案：能，但需要改进

Omni：生产系统（Production System）
  → "如何构建稳定可靠的比特币资产协议？"
  → 答案：完整的协议栈 + 明确的规范

时间跨度：2012-2014
演进路径：实验 → 改进 → 成熟
```

彩色币的探索为比特币资产协议开辟了道路，但真正让这条道路可行的，是 Omni 带来的工程化和规范化。

**下一章，我们将深入 Omni Layer，理解第一个"生产级"比特币资产协议是如何设计的。**



---

## 3.10 彩色币在本书中的位置

彩色币代表了比特币数据嵌入史的**"第 0 代协议"**。

### 协议演进链条

```
彩色币的教训 → 后续协议的改进

├─ 缺乏明确数据 → Omni 引入 OP_RETURN 指令
├─ 缺乏结构化 → Counterparty 引入结构化 payload
├─ 缺乏承诺 → RGB 引入 Taproot commitment
├─ UTXO 污染 → Ordinals 使用 witness（不污染 UTXO 集）
└─ 钱包兼容性 → 所有后续协议都设计专用索引器
```

### 技术贡献总结

| 贡献 | 影响 |
|------|------|
| Off-chain 状态机 | RGB、Client-Side Validation |
| UTXO 携带资产 | 所有后续协议的基础 |
| Order-based coloring | Atomicals 的 ARC-20 |
| Genesis 交易概念 | 通用的资产发行模型 |
| 暴露零数据方案的局限 | 推动 OP_RETURN 标准化 |

---

## 3.11 小结：彩色币的工程结论

**彩色币告诉我们：**

✅ **可行性**：在完全不写（或少写）字节的前提下，确实能构建资产协议  
✅ **链外状态**：off-chain 状态机是可行的技术路线  
✅ **UTXO 抽象**：UTXO 可以承载比特币之外的资产语义

❌ **不可靠性**：隐式协议永远无法保持稳定  
❌ **生态挑战**：钱包行为会直接破坏协议  
❌ **表达能力**：缺乏明确数据结构无法支持复杂应用  
❌ **经济成本**：Dust limit 让 EPOBC 的小额资产不可行

**历史地位：**

彩色币虽然失败了，但它是整个比特币资产协议历史的开端。

它的两个关键贡献：
1. **证明了 UTXO 模型可以承载资产**
2. **引入了 OP_RETURN 作为数据容器**（Open Assets）

后续的所有协议——Omni、Counterparty、Stamps、Ordinals、Atomicals、RGB——都是在彩色币的基础上演进而来。

---

## 📚 延伸学习

### 实践部分

本章专注于历史和原理。对于想要深入代码实现的读者，请参考：

**[Chapter 3 Part 2: 彩色币协议动手实践](./ch03_colored-coins_动手实践.md)**

动手实践包含：
- EPOBC 的脆弱性演示
- Open Assets 的 OP_RETURN 实现
- 攻击场景的代码复现
- 对比实验

### 参考资源

**核心文献：**
- [Colored Coins Whitepaper (Meni Rosenfeld, 2012)](https://github.com/bitcoinx/colored-coins)
- [Open Assets Protocol Specification](https://github.com/OpenAssets/open-assets-protocol)
- [EPOBC on Bitcoin Wiki](https://en.bitcoin.it/wiki/Colored_Coins)

**历史资料：**
- [Bitcointalk: Colored Coins 讨论帖 (2012)](https://bitcointalk.org/index.php?topic=106373.0)
- [Flavien Charlon 的 Open Assets 博客](https://github.com/OpenAssets/open-assets-protocol/blob/master/specification.mediawiki)

**开源实现：**
- [Coinprism](https://github.com/Coinprism/openassets) - Open Assets 参考实现
- [ChromaWay](https://github.com/chromaway/ngcccbase) - EPOBC 实现

---

**下一章预告：**

**第 4 章：Omni Layer（Mastercoin）—— 第一个生产级协议**

- OP_RETURN 的成熟规范
- USDT 的诞生故事
- 从实验到生产的跨越

让我们看看彩色币的教训如何催生了真正可用的比特币资产协议。