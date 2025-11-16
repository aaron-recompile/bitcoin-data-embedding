# 人们如何尝试将非交易数据推入比特币

*比特币上所有数据嵌入技术的可复现工程史*

**English / [English Version](../../README.md)**

---

## 概述

本仓库系统性地、历史性地、以可复现的工程细节，记录了人们在过去十五年中如何尝试将非交易数据嵌入比特币。

比特币并非设计为通用数据层。
然而，开发者、协议设计者、艺术家和金融工程师一直在尝试存储：

- 元数据
- 图像
- 指令和脚本
- 资产状态
- Merkle 承诺
- 任意二进制数据

……在比特币严格的 UTXO 和脚本模型中。

本项目是首次系统性尝试：

- 重建比特币上使用过的所有主要数据嵌入技术
- 解释每种技术的结构原理
- 提供可复现的测试网实现以供验证

---

## 动机

比特币的共识规则并未暴露显式的"数据字段"。
然而，其结构包含可以注入数据的隐式缝隙：

- **输入**（scriptSig、witness、script path）
- **输出**（scriptPubKey、OP_RETURN、Taproot leaf 承诺）

每个协议——彩色币、Omni、Counterparty、Stamps、Ordinals、Atomicals、Runes、RGB——本质上都是利用这些结构缝隙的不同方式。

本仓库旨在：

- 澄清共识与策略的边界
- 展示比特币实际允许的内容，而非人们的假设
- 记录所有已知的数据嵌入工程模式
- 提供可复现代码，让读者可以在测试网上运行协议
- 为协议研究者提供清晰的参考，避免重复过去的错误
- 帮助比特币开发者社区在上下文中评估未来提案

**这不是一份推广文档。**
**这是一部技术史，从第一性原理重建。**

---

## 仓库结构

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

## 内容摘要

每章解释：

- 历史背景和动机
- 精确的工程机制
- 原始交易分解
- Witness / script path 分析
- 协议如何融入比特币数据模型
- 失败模式和长期约束
- 后续协议继承或修正的内容

### 章节列表

1. **序言** — 为什么人们想在比特币上存储非交易数据
2. **数据入口完整地图** — 输入与输出结构分析
3. **彩色币** — 零字节协议和隐式状态机
4. **Omni (Mastercoin)** — OP_RETURN 的首次正式使用
5. **Counterparty** — 通过 OP_RETURN + 多签技术嵌入数据
6. **Stamps** — 极端的裸多签数据载荷
7. **P2WSH** — 当脚本变成结构化数据容器时
8. **Ordinals** — 任意 witness 数据和"铭文"
9. **Atomicals** — 基于 Taproot 的对象编码
10. **Runes** — 现代 OP_RETURN 资产模型
11. **RGB** — 客户端验证和链下状态承诺
12. **未来方向** — 承诺、契约和数据层边界

每章包括测试网交易、原始十六进制转储和可复现代码。

---

## 可复现性

书中的所有示例都可以使用以下工具完全复现：

- Python 3
- `bitcoinlib` / `bitcoinutils`
- Bitcoin Core（regtest 或 testnet）
- 简单的命令行脚本

### 快速开始

```bash
cd code/omni/
python3 create_opreturn_tx.py
```

或

```bash
cd code/ordinals/
python3 inscribe.py
```

每个代码目录包括：

- 带设置说明的 `README.md`
- 原始示例交易
- 带注释的 witness/script 转储
- 解释数据位置的图表

---

## 设计原则

本项目建立在四个核心原则之上：

### 1. 可复现性

每个声明都得到可在链上验证的真实原始交易支持。

### 2. 中立性

不推广也不批评任何协议。
每个协议都严格按照工程价值和技术准确性进行评估。

### 3. 边界清晰

明确区分：

- 共识规则（比特币节点必须接受的内容）
- 策略规则（Bitcoin Core 选择中继的内容）
- 钱包行为（实现特定的）
- 解释层（协议特定的解析）

### 4. 最小假设

所有解释都来自比特币的实际数据结构，而非推测性叙述或营销声明。

---

## 贡献

欢迎提交 Pull Request 和 Issue。

我们特别感谢：

- 对历史细节和时间线的修正
- 带有原始交易十六进制的额外测试网示例
- Script/path 分解和可视化
- 图表或视觉辅助
- 来自长期比特币贡献者的策略和历史背景

这旨在成为比特币开发者社区的协作技术参考。

---

## 许可证

- **书籍文本**：知识共享署名-相同方式共享 4.0 国际许可协议 (CC BY-SA 4.0)
- **代码**：MIT 许可证

---

## 其他格式

本项目还提供：

- **GitBook** — 带交互式图表的可视化阅读体验
- **Medium 系列** — 逐章文章
- **Leanpub 书籍** — PDF/ePub 格式（免费或按需付费）

每种格式上线后，链接将在此处添加。

---

## 联系方式

讨论、修正或合作：

- **Twitter**: [@aaron_recompile](https://twitter.com/aaron_recompile)
- **GitHub Issues**: 技术问题和修正

---

## 结语

比特币的数据嵌入历史并非奇闻轶事——
它是理解以下内容的关键：

- 协议设计约束
- 策略辩论及其技术根源
- 费用市场演变
- 比特币作为可编程结算层的未来

本仓库从构建者的角度，而非投机者的角度，记录这段历史。

**这里讨论的每个协议都代表了对比特币边界的真实尝试。**
**理解它们如何工作——以及为什么有些成功而有些失败——对于任何在比特币上构建的人来说都至关重要。**

