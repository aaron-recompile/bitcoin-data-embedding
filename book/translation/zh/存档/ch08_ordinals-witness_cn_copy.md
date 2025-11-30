# 第 8 章：Ordinals —— Taproot 信封与数据层的爆发

## 从 witness 容器到社会现象

上一章我们看到，P2WSH 把脚本从 scriptSig 移到 witness，
带来了 75% 的成本折扣和更清晰的结构分离。

但真正引爆"witness 作为数据层"的，是 2021 年 11 月激活的 **Taproot 升级**。

Taproot 做了一件关键的事：

**移除了 witness 数据的大小上限。**

在 Taproot 之前，单个 witness 栈元素有 520 字节的限制。
Taproot 之后，只要整个区块不超过 4MB（约 4 million weight units），
单笔交易的 witness 理论上可以占满整个扩展区块。

这为"把任意大数据塞进比特币"打开了大门。

2023 年 1 月，Casey Rodarmor 发布了 **Ordinals 协议**，
它不是第一个利用 witness 的协议，
却是第一个将这一能力**推到社会层面引爆争议**的协议。

本章将解析：

- Ordinals 的两步设计：聪排序 + 内容铭刻
- commit-reveal 模式与 OP_FALSE OP_IF 信封
- 为什么 Taproot script path 是理想的数据容器
- 工程复现：构造一个 Ordinals 风格的铭刻
- 争议与边界
- 章末：Atomicals 等变体协议的简要介绍

---

## 8.1 Ordinals 的第一步：给每一个 satoshi 排号

Ordinals 先不谈数据，只做一件事：

**对所有 satoshi 建立一个确定的排序与编号规则。**

核心规则：**先进先出（FIFO）**

```
区块 1 → 区块 2 → 区块 3 → ...
  ↓        ↓        ↓
交易 1 → 交易 2 → 交易 3 → ...
  ↓        ↓        ↓
Input → Output → Input → Output → ...
  ↓        ↓
按金额拆分成 sat，按顺序编号
```

这样，理论上每一个 sat 都可以被赋予一个"序号"（ordinal number），
范围从 0 到 2,100,000,000,000,000（2.1 千万亿）。

例如：

> "这个 sat 的序号是 1,234,567,890，最早出现在区块 100,000 的 coinbase，
> 后来经历了地址 A → B → C。"

**关键洞察：**

这一步完全在**解释层**完成，不涉及共识。
比特币协议本身并不知道"ordinal number"的存在，
这只是一个外部观察者对 sat 流动的追踪规则。

---

## 8.2 第二步：把数据"铭刻"在 witness 里

Ordinals 的第二步：

**在承载某个 sat 的交易的 witness 里，写入一段数据，定义为"铭刻"（inscription）。**

### 8.2.1 commit-reveal 两阶段模式

铭刻采用**两阶段**流程：

**第一阶段：Commit（承诺）**

创建一个 Taproot 输出（P2TR），其内部承诺了一个包含铭刻数据的脚本。

```
scriptPubKey: OP_1 <32-byte tweaked pubkey>
```

此时，铭刻内容还没有暴露在链上。
外部观察者只能看到一个普通的 Taproot 地址。

**第二阶段：Reveal（揭示）**

花费上述 Taproot 输出，通过 script path 揭示铭刻内容。

```
witness:
  <signature>
  <inscription script>
  <control block>
```

铭刻数据在 `<inscription script>` 中，此时才真正"上链"。

### 8.2.2 为什么用 commit-reveal？

1. **防止抢跑（front-running）**：如果直接在一笔交易里暴露铭刻内容，
   矿工或其他观察者可能抢先复制相同内容。commit-reveal 确保内容在确认前不可见。

2. **优化费用**：commit 阶段的输出只有 32 字节哈希，
   大数据在 reveal 阶段才出现，此时享受 witness 折扣。

3. **符合 Taproot 设计**：Taproot script path 本身就是"承诺-揭示"结构。

---

## 8.3 OP_FALSE OP_IF 信封：让数据"不执行"

铭刻数据被包裹在一个特殊的"信封"（envelope）结构里：

```
OP_FALSE
OP_IF
  OP_PUSH "ord"
  OP_PUSH 1
  OP_PUSH <content-type>
  OP_PUSH 0
  OP_PUSH <data chunk 1>
  OP_PUSH <data chunk 2>
  ...
OP_ENDIF
```

### 8.3.1 为什么这样设计？

**OP_FALSE OP_IF ... OP_ENDIF** 的作用：

1. `OP_FALSE` 把 0 压入栈
2. `OP_IF` 检查栈顶，发现是 0（false），跳过整个 IF 块
3. 块内的 PUSHDATA 指令**从不执行**
4. `OP_ENDIF` 结束条件块
5. 脚本继续执行后面的验证逻辑（通常是 `<pubkey> OP_CHECKSIG`）

**结果：**

- 数据被"塞进"脚本，但不参与执行
- 不占用栈空间
- 不影响脚本验证结果
- 从比特币节点的角度，这是一段"合法但无用"的脚本数据

### 8.3.2 完整的铭刻脚本示例

一个典型的铭刻脚本：

```
OP_FALSE
OP_IF
  OP_PUSH "ord"           # 协议标识
  OP_PUSH 1               # 表示下一个 push 是 content-type
  OP_PUSH "text/plain"    # MIME 类型
  OP_PUSH 0               # 表示下面是内容
  OP_PUSH "Hello, Ordinals!"
OP_ENDIF
<x-only pubkey>
OP_CHECKSIG
```

执行流程：

1. `OP_FALSE` 压入 0 到栈
2. `OP_IF` 检查栈顶是 0（false），跳过整个 IF 块
3. IF 块内的所有 PUSHDATA 指令**从不执行**，但数据保留在脚本字节中
4. `<pubkey> OP_CHECKSIG` 验证 witness 提供的签名 → 返回 TRUE
5. 栈顶是 TRUE，脚本通过

**关键点：数据保留在脚本字节里，但从不参与计算。**

---

## 8.4 Taproot script path：理想的数据容器

### 8.4.1 为什么 Taproot 而不是 P2WSH？

| 特性 | P2WSH | Taproot (P2TR) |
|------|-------|----------------|
| witness 元素大小限制 | 520 bytes | **无单元素大小限制** |
| 脚本大小限制 | 10,000 bytes | **无脚本大小限制**（仅受区块限制） |
| 结构 | 单一 witnessScript | **Merkle 树**（多 leaf） |
| 隐私 | 揭示完整脚本 | 只揭示使用的 leaf |
| 费用 | 1 WU/byte | 1 WU/byte |

**Taproot 的关键突破：**

BIP 342（Tapscript）移除了：
- 单个 push 的 520 字节限制
- 脚本总大小的 10,000 字节限制

现在，只要整个区块的 weight 不超过 4,000,000 WU（约 4MB），
单笔交易的 witness 可以任意大。

**技术说明：**
虽然 Ordinals 理论上也可以在 P2WSH 上实现（受 520 字节元素限制），
但实际实现都选择 Taproot，因为 Taproot 移除了大小限制，
更适合存储大文件（如图片、视频等）。

### 8.4.2 Taproot 内部结构回顾

```
                    [Tweaked Public Key]
                           |
                    [Internal Key] + [Merkle Root]
                                          |
                                   ┌──────┴──────┐
                                [Leaf A]      [Leaf B]
                                   |
                         OP_FALSE OP_IF
                           <inscription>
                         OP_ENDIF
                         <pubkey> OP_CHECKSIG
```

铭刻数据被放在某个 leaf script 里，
通过 control block 和 Merkle proof 在花费时揭示。

---

## 8.5 工程复现：构造一个简化版铭刻

构造 Ordinals 风格铭刻的核心流程：

**第一阶段：Commit（承诺）**

1. 生成密钥对，构造铭刻脚本（包含 OP_FALSE OP_IF 信封）
2. 将脚本放入 Taproot leaf，计算 Merkle root
3. 用 internal pubkey 和 Merkle root 计算 tweaked pubkey
4. 生成 P2TR 地址，发送 BTC 到该地址

此时，铭刻内容尚未暴露，外部只能看到普通的 Taproot 输出。

**第二阶段：Reveal（揭示）**

1. 花费上述 Taproot 输出
2. 在 witness 中提供：
   - signature（对交易的签名）
   - inscription_script（完整的铭刻脚本）
   - control_block（Merkle 证明）
3. 广播交易，铭刻内容正式上链

**解析铭刻**

从链上交易的 witness 中提取铭刻脚本，解析 OP_FALSE OP_IF 块内的数据：
- 识别 "ord" 协议标记
- 提取 content-type 和 content

通过这个流程，读者可以理解：

**铭刻不是魔法，就是 witness 里的字节 + 一个解释器。**

> **注：** 完整的可运行代码示例请参考配套代码仓库 `code/ordinals/` 目录。

---

## 8.6 数据流向与 UTXO 关系

### 8.6.1 铭刻不污染 UTXO 集

这是 Ordinals 相比 Stamps 的关键优势：

| 方式 | 数据位置 | 对 UTXO 集的影响 |
|------|---------|-----------------|
| Stamps (裸多签) | Output scriptPubKey | **永久占用** UTXO 集 |
| Ordinals | Input witness | **不影响** UTXO 集 |

**原因：**

- Stamps 把数据放在**输出**的 scriptPubKey 里（伪公钥），
  这些输出无法被花费，永久留在 UTXO 集中。

- Ordinals 把数据放在**输入**的 witness 里，
  一旦交易确认，witness 数据**保存在区块历史中**（确实上链），
  但不进入 UTXO 集。

### 8.6.2 数据存储的位置

```
区块结构：
├── Block Header
├── Transactions
│   ├── Tx 1
│   │   ├── Inputs
│   │   │   ├── prevout
│   │   │   └── witness  ← 铭刻数据在这里
│   │   └── Outputs
│   └── Tx 2 ...
└── ...

UTXO 集：
├── Output A (可花费)
├── Output B (可花费)
└── ...  ← 不包含 witness 数据
```

**运行全节点的成本：**

- UTXO 集需要常驻内存/SSD，大小敏感
- 区块历史可以存在机械硬盘，大小不那么敏感
- Ordinals 增加了区块历史的大小，但不增加 UTXO 集负担

**重要澄清：**
铭刻数据确实保存在链上（区块历史中），任何拥有完整区块历史的节点都可以访问。
区别在于：UTXO 集需要常驻内存/SSD 以支持快速查询，而区块历史可以存储在机械硬盘上，
对节点性能影响较小。但这不意味着数据"不在链上"，只是存储位置和访问方式不同。

### 8.6.3 铭刻的"持久性"问题

**Pruned Node 的挑战：**

运行 pruned 节点的用户（只保留最近 N 个区块）可能无法访问历史铭刻数据。
这是因为：

- Pruned 节点会删除旧区块数据以节省空间
- 铭刻数据存储在区块历史的 witness 中
- 一旦区块被删除，铭刻数据就不可访问了

**数据保证的层次：**

| 节点类型 | 能否访问历史铭刻 | 数据保证 |
|---------|----------------|---------|
| 完整节点 | ✅ 是 | 完整历史数据 |
| Pruned 节点 | ❌ 否（旧区块已删除） | 仅保留最近 N 个区块 |
| 轻节点 | 依赖索引器 | 依赖第三方服务 |

**关键洞察：**

Ordinals 的"数据上链"依赖于**完整节点保留完整区块历史**。
对于 pruned 节点用户，铭刻数据可能不可访问，这暴露了 witness 数据存储的"持久性"问题：
数据确实在链上，但访问需要完整的历史数据。

---

## 8.7 争议：创新还是 spam？

Ordinals 引发了比特币社区最激烈的争论之一。

### 8.7.1 支持方观点

1. **协议允许**：比特币脚本本就允许任意数据，这是自由使用的体现
2. **付费使用**：铭刻用户为 block space 支付了真金白银的手续费
3. **witness 定价**：witness 数据享受 75% 折扣，这是协议设计的结果
4. **增加需求**：铭刻带来了新的区块空间需求，提高矿工收入
5. **扩展用例**：展示了比特币作为数据层的可能性

### 8.7.2 反对方观点

1. **偏离初衷**：比特币是"点对点电子现金系统"，不是图床
2. **推高费用**：大规模铭刻导致普通交易费用飙升
3. **节点负担**：区块膨胀增加了同步和存储成本
4. **垃圾数据**：大量铭刻内容是低质量的投机 NFT
5. **policy 收紧**：可能推动 Core 收紧标准规则，影响其他用例

### 8.7.3 Core 开发者的回应

部分 Core 开发者提议收紧 `datacarriersize` 等策略规则，
但这只是节点的**转发策略**，不是共识规则。
矿工仍然可以打包任何符合共识的交易。

Luke Dashjr 曾提交 PR 试图过滤铭刻交易，但未被合并。
最终社区达成的粗略共识是：

> "不喜欢，但无法阻止。协议层不应该审查合法交易。"

### 8.7.4 本书立场

本书不站队，只从结构和工程角度分析：

- Ordinals **在技术上是合法的**：它使用的都是标准操作码和结构
- 它**暴露了 witness 折扣的设计后果**：便宜的存储空间会被利用
- 它**引发了关于比特币边界的讨论**：货币系统 vs 通用数据层

这些讨论对理解比特币的本质是有价值的。

---

## 8.8 Ordinals 在数据嵌入史中的位置

| 时代 | 协议 | 数据位置 | 特点 |
|------|------|---------|------|
| 创世 | Coinbase | Coinbase | 仅矿工可用 |
| OP_RETURN | 各种协议 | Output | 80 字节限制 |
| 多签滥用 | Counterparty, Stamps | Output (伪公钥) | 污染 UTXO |
| SegWit | P2WSH | Witness | 520 字节元素限制 |
| **Taproot** | **Ordinals** | **Witness** | **无大小限制，社会爆发** |

Ordinals 标志着：

1. **witness 时代数据嵌入的正式爆发**
2. **从协议工程师的玩具变成普通用户的炒作对象**
3. **script/witness 第一次大规模进入公众视野**

---

## 8.9 变体协议：Atomicals、BRC-20 及其他

Ordinals 打开的大门，催生了一系列变体协议。
它们共享相同的底层机制，但在上层设计上各有侧重。

### 8.9.1 Atomicals：聪染色 + 结构化对象

**发布时间**：2023 年 9 月

**核心差异**：

| 维度 | Ordinals | Atomicals |
|------|----------|-----------|
| 信封标记 | `"ord"` | `"atom"` |
| 聪处理 | 给聪编号（ordinal number） | 给聪"染色"（colored coin） |
| 数据模型 | 内容（图片/文本） | 结构化对象（Asset/NFT/Realm） |
| 代币标准 | BRC-20（JSON 铭刻） | ARC-20（聪背书） |
| 额外功能 | 无 | **Bitwork 挖矿**（可选 PoW） |

**Bitwork 挖矿**（工程优化亮点）：

Atomicals 引入了可选的 PoW 机制，要求铸造交易的 TXID 必须以特定前缀开头。
通过调整 witness 中的 nonce 字段，不断重新计算交易哈希，直到满足要求。
这增加了铸造成本，防止 spam，并为"稀有度"提供客观度量。

> **注：** Bitwork 挖矿的具体实现细节和工程设计思路，详见附录或补充章节。

**ARC-20 vs BRC-20**：

| 维度 | BRC-20 | ARC-20 |
|------|--------|--------|
| 数据格式 | JSON 铭刻 | CBOR 编码 |
| 代币单位 | 任意（JSON 定义） | 1 token = 1 sat |
| 转账逻辑 | 链下索引解析 JSON | **原生 UTXO 规则** |

ARC-20 的 "1 token = 1 sat" 设计使得代币转账直接遵循比特币的 UTXO 规则，
无需复杂的链下索引。

### 8.9.2 BRC-20：JSON 铭刻代币

**发布时间**：2023 年 3 月

BRC-20 使用 Ordinals 铭刻 JSON 来定义代币：

```json
{
  "p": "brc-20",
  "op": "deploy",
  "tick": "ordi",
  "max": "21000000",
  "lim": "1000"
}
```

**问题**：

- 代币状态完全依赖链下索引器解析
- 不同索引器可能产生不同结果
- 与比特币原生 UTXO 模型脱节

### 8.9.3 Runes：Casey Rodarmor 的"修正"

**发布时间**：2024 年 4 月（比特币第四次减半时）

Ordinals 创始人 Casey Rodarmor 后来推出 **Runes** 协议，
试图用更简洁的方式实现代币功能：

- 回归 OP_RETURN（而非 witness）
- 代币状态直接绑定 UTXO
- 无需复杂索引器

这是对 BRC-20 乱象的回应，下一章将详细讨论。

### 8.9.4 变体协议对比总结

| 协议 | 数据位置 | 信封/格式 | 核心创新 |
|------|---------|----------|---------|
| Ordinals | Witness | `"ord"` + MIME | 聪编号 + 内容铭刻 |
| Atomicals | Witness | `"atom"` + CBOR | 聪染色 + Bitwork 挖矿 |
| BRC-20 | Witness | JSON | 链下索引代币 |
| Runes | OP_RETURN | 二进制 | UTXO 原生代币 |

**共同点**：

- 都利用 Taproot 后的 witness 容量
- 都采用 commit-reveal 模式
- 都依赖链下解释器赋予数据"意义"

**关键洞察**：

> 这些协议的底层机制几乎相同，
> 差异在于上层的**数据模型**和**社会叙事**。

---

## 8.10 小结：Taproot 信封的范式

Ordinals 及其变体展示了一个通用范式：

```
┌─────────────────────────────────────────────┐
│           Taproot 信封范式                    │
├─────────────────────────────────────────────┤
│  1. Commit：创建 P2TR 输出，承诺脚本哈希        │
│  2. Reveal：花费输出，揭示 script path         │
│  3. 信封：OP_FALSE OP_IF <data> OP_ENDIF     │
│  4. 解释：链下索引器解析数据，赋予语义           │
└─────────────────────────────────────────────┘
```

**工程意义**：

- Taproot 提供了结构（Merkle tree、script path）
- OP_FALSE OP_IF 提供了"不执行的数据容器"
- commit-reveal 提供了隐私和防抢跑
- witness 折扣提供了经济激励

**社会意义**：

- 第一次让普通用户意识到比特币可以存储任意数据
- 引发了关于比特币本质的深层讨论
- 为后续协议（RGB、Runes 等）提供了对比参照

下一章，我们将看到一个"回归"的尝试：

**第 9 章：Runes —— 用 OP_RETURN 重新定义代币**

Casey Rodarmor 如何用更简洁的设计，试图修正 BRC-20 的混乱。