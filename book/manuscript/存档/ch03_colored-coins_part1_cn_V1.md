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

## 3.4 EPOBC 协议核心机制

### 3.4.1 nSequence 零开销标记

EPOBC 的天才之处在于利用了交易输入中已存在的 nSequence 字段：

**标记方案：**

```python
# nSequence 低 6 位编码交易类型
0x25 = 37 = 0b100101  → Genesis (发行)
0x33 = 51 = 0b110011  → Transfer (转账)

# 检测逻辑
tag = tx.inputs[0].sequence & 0x3F  # 取低6位

if tag == 0x25:
    # Genesis 交易：第一个输出是彩色币
    color_id = f"{txid}:0"
    quantity = output[0].value  # 金额=数量
elif tag == 0x33:
    # Transfer 交易：order-based coloring
    # 按输出顺序分配颜色
```

**关键特性：**

- ✅ 零字节开销（nSequence 本来就存在）
- ✅ 不增加交易大小
- ✅ 对矿工完全透明
- ❌ 资产数量与 satoshi 耦合
- ❌ 缺乏元数据支持

> **💡 动手实践：**  
> 完整的 EPOBC 实现代码、详细注释和测试请参考动手实践部分 Section 1。

### 3.4.2 Order-Based Coloring 算法

**核心原理：**

```
输入序列 → 输出序列的映射

Input:  [UTXO_1: 100 units]
        
         ↓ Order-Based Coloring
        
Output: [30 sats → 30 units,
         70 sats → 70 units]

规则：
  quantity(output_i) = min(output_i.value, remaining)
```

这个算法简单但脆弱，任何钱包行为（合并输入、调整顺序）都会破坏颜色传播。

---

## 3.5 Open Assets Protocol 核心机制

### 3.5.1 Marker Output 结构

Open Assets 使用 OP_RETURN 创建"marker output"来存储资产数量信息：

**数据格式：**

```
6a                     # OP_RETURN
<length>               # Payload 长度
  4f 41                # "OA" 魔数
  01 00                # Version 1
  <count>              # 资产数量个数
  <qty1> <qty2> ...    # LEB128 编码的数量列表
  <meta_len>           # 元数据长度
  <metadata>           # 可选元数据
```

**实际示例（[300, 700]）：**

```
6a 0a 4f 41 01 00 02 ac 02 bc 05 00

解析：
6a        → OP_RETURN
0a        → 10 bytes
4f 41     → "OA"
01 00     → Version 1
02        → 2 个数量
ac 02     → 300 (LEB128: 0xac 0x02)
bc 05     → 700 (LEB128: 0xbc 0x05)
00        → 无元数据
```

**LEB128 编码关键点：**

```python
# 300 的编码
300 = 0b100101100

第1字节: 低7位 = 0b0101100 = 0x2C
         还有剩余 → 设置延续位 → 0xAC
第2字节: 低7位 = 0b0000010 = 0x02
         无剩余 → 完成

结果: ac 02
```

> **💡 动手实践：**  
> LEB128 编码的详细实现、编码表、交互式演示请参考动手实践部分 Section 2.2。

### 3.5.2 Asset ID 计算

Open Assets 的 Asset ID 通过双哈希生成：

```python
Asset ID = RIPEMD160(SHA256(scriptPubKey))
```

**为什么这样设计？**

1. **全局唯一性**：SHA256 几乎不可能碰撞
2. **发行者控制**：只有拥有私钥的人才能创建特定 Asset ID
3. **压缩长度**：RIPEMD160 压缩到 20 bytes（与比特币地址一致）
4. **可验证性**：任何人都可以验证 Asset ID 来自特定地址

**示例：**

```python
# P2PKH 发行地址
script = b'\x76\xa9\x14' + pubkey_hash + b'\x88\xac'

# Step 1: SHA256
sha256 = hashlib.sha256(script).digest()
# → 32 bytes

# Step 2: RIPEMD160
asset_id = hashlib.new('ripemd160').update(sha256).digest()
# → 20 bytes (160 bits)
# → 例如: 3903d5b563d3e8c76d62ec31bcbeb68ee1e651af
```

> **💡 动手实践：**  
> Asset ID 的完整实现、不同脚本类型的测试请参考动手实践部分 Section 2.4。

### 3.5.3 Order-Based Coloring 算法

Open Assets 的颜色传播规则：

**核心逻辑：**

```
1. 收集所有输入的资产数量
   total_input = Σ input_quantities

2. 从 marker 读取输出数量列表
   output_quantities = marker.quantities

3. 验证守恒
   Σ output_quantities ≤ total_input

4. 按顺序分配
   for i, qty in enumerate(output_quantities):
       output[marker_index + 1 + i] → qty units
```

**关键特性：**

- ✅ 数量与 BTC 金额解耦
- ✅ 支持资产销毁（输出 < 输入）
- ✅ Marker 明确标识资产交易
- ❌ 仍然依赖输出顺序
- ❌ 不支持原子交换（不能混合不同资产）

> **💡 动手实践：**  
> Open Assets Parser 的完整实现、测试用例请参考动手实践部分 Section 2.3 和 2.5。

---

## 3.6 彩色币的致命缺陷

### 3.6.1 技术缺陷

**1. 钱包行为不可控**

普通钱包会：
- 自动合并多个输入（UTXO consolidation）
- 自动调整输出顺序
- 添加找零地址
- 使用 RBF（Replace-By-Fee）

这些行为会立即破坏颜色传播链条。

```python
# 演示：钱包合并破坏颜色

# 用户持有：
# UTXO_1: 0.0001 BTC + 100 ColorA
# UTXO_2: 0.0005 BTC (普通 BTC)

# 钱包自动构造交易：
tx = {
    'inputs': [
        UTXO_1,  # 100 ColorA
        UTXO_2   # 普通 BTC
    ],
    'outputs': [
        {'value': 0.0003, 'addr': 'recipient'},
        {'value': 0.0003, 'addr': 'change'}
    ]
}

# 问题：
# - 100 ColorA 应该分配给哪个输出？
# - 按金额分？按顺序分？
# - 普通 BTC 会"稀释"颜色吗？
# - 不同钱包的解释不同 → 颜色丢失
```

**2. 缺乏原子性保证**

```python
# 资产交换场景
# Alice 想用 50 ColorA 换 Bob 的 0.01 BTC

# 问题：
# - 如何保证同时交换？
# - 没有脚本级别的原子性
# - 必须依赖链外协调
# - 容易被攻击
```

**3. Anti-Dust Patch 的打击**

2013年4月，Bitcoin Core 引入 anti-dust patch，规定最小输出为 5,430 satoshis（约 0.0000543 BTC）。

这对彩色币是重大打击：

```python
# 之前可以：
output = {
    'value': 1,  # 1 satoshi = 1 个资产单位
    'colored': True
}

# 之后必须：
output = {
    'value': 5430,  # 最小 5,430 satoshis
    'colored': True  # 但仍然只代表 1 个资产单位
}

# 问题：
# - 发行成本暴涨 5000 倍
# - 小额资产转账不再可行
# - UTXO 集污染严重
```

**4. 缺乏元数据表达能力**

早期 EPOBC 协议：

```python
# 只能表达：
colored_coin = {
    'quantity': 100,
    'color_id': 'tx1:0'
}

# 无法表达：
# - 资产名称
# - 发行者信息
# - 可分割性
# - 过期时间
# - 转让限制
# - 智能合约逻辑
```

### 3.6.2 攻击向量

```python
class ColoredCoinAttacks:
    """
    彩色币协议的攻击示例
    """
    
    @staticmethod
    def color_washing():
        """
        攻击1：颜色洗白
        混入大量普通 BTC 输入，破坏颜色映射
        """
        attack_tx = {
            'inputs': [
                {'colored': True, 'quantity': 100, 'color': 'A'},
                {'colored': False, 'value': 100000}  # 大量普通 BTC
            ],
            'outputs': [
                {'value': 50000},  # 颜色映射混乱
                {'value': 50100}
            ]
        }
        # 结果：100 ColorA 被"洗白"成普通 BTC
    
    @staticmethod
    def output_reordering():
        """
        攻击2：输出重排序
        改变输出顺序，改变颜色分配
        """
        # 合法交易
        tx1 = {
            'inputs': [{'colored': 100, 'color': 'A'}],
            'outputs': [
                {'value': 30, 'addr': 'alice'},   # 30 ColorA
                {'value': 70, 'addr': 'attacker'} # 70 ColorA
            ]
        }
        
        # 攻击者重排（Replace-By-Fee）
        tx2 = {
            'inputs': [{'colored': 100, 'color': 'A'}],
            'outputs': [
                {'value': 70, 'addr': 'attacker'}, # 70 ColorA
                {'value': 30, 'addr': 'alice'}     # 30 ColorA
            ]
        }
        # 如果钱包按输出顺序分配颜色 → Alice 获得量改变
    
    @staticmethod
    def dust_attack():
        """
        攻击3：Dust 攻击
        创建大量小额彩色 UTXO，污染 UTXO 集
        """
        for i in range(1000):
            dust_tx = {
                'inputs': [{'colored': 1000}],
                'outputs': [
                    {'value': 600, 'colored': 1}  # 1 个单位
                    for _ in range(1000)
                ]
            }
        # 结果：UTXO 集膨胀，节点资源耗尽
```

---

## 3.7 为什么彩色币最终失败？

总结其失败的根本原因：

### (1) 缺乏强制执行机制

彩色币技术无法阻止用户以破坏额外信息的方式操作底层比特币，因为彩色币操作受比特币交易规则约束，而彩色币的要求更严格但不被网络强制执行。

```python
# 比特币共识层不关心颜色
class BitcoinConsensus:
    def validate_transaction(self, tx):
        # ✅ 检查签名
        # ✅ 检查输入金额 >= 输出金额
        # ✅ 检查 double-spend
        # ❌ 不检查颜色是否正确传播
        # ❌ 不检查资产数量是否守恒
        pass
```

### (2) 钱包生态不兼容

由于彩色币是使用多种不同算法实现的，不同钱包之间的交易可能导致货币着色特性丢失，彩色币需要能够区分非比特币项目的统一钱包。

```python
# 需要专门的彩色币钱包
color_aware_wallets = [
    'Coinprism',  # Open Assets
    'ChromaWallet',  # EPOBC
    'Colu',  # Colu Protocol
    'CoinSpark'  # CoinSpark Protocol
]

# 普通钱包完全不知道颜色
regular_wallets = [
    'Bitcoin Core',
    'Electrum',
    'Any other wallet'
]

# 问题：
# - 用户必须使用专门钱包
# - 不同彩色币协议不兼容
# - 一旦发到普通钱包 → 颜色永久丢失
```

### (3) 缺乏链上承诺（commitment）

```python
# 彩色币完全没有使用：
commitments = {
    'OP_RETURN': '很少或不使用（EPOBC）',
    'witness': '不存在（2013年还没有 SegWit）',
    'Taproot': '不存在（2021年才有）',
    'Merkle commitment': '不存在',
    'Cryptographic proof': '几乎没有'
}

# 结果：
# - 状态无法被独立验证
# - 需要完整扫描区块链
# - SPV 客户端支持差
# - 无法证明资产所有权
```

### (4) 经济成本问题

```python
import math

# Anti-dust 之前
def cost_before_antidust():
    units = 1000000  # 发行100万个单位
    satoshis_per_unit = 1
    total_btc = units * satoshis_per_unit / 100000000
    print(f"成本: {total_btc} BTC")
    # 成本: 0.01 BTC

# Anti-dust 之后
def cost_after_antidust():
    units = 1000000
    satoshis_per_unit = 5430  # 最小输出
    total_btc = units * satoshis_per_unit / 100000000
    print(f"成本: {total_btc} BTC")
    # 成本: 54.3 BTC (!)
    
    # 如果 BTC = $40,000
    print(f"美元成本: ${total_btc * 40000}")
    # 美元成本: $2,172,000

cost_after_antidust()
```

---

## 3.8 彩色币的历史贡献

尽管彩色币失败了，但它的思想在后代中不断被继承：

### ① 提出了"off-chain 状态机"模型

RGB、Taproot Assets 等现代协议使用客户端验证（Client-Side Validation）的思想，将状态保存在链外，链上只做最小承诺。

```python
# 彩色币思想
class ColoredCoins:
    state = "链外解释"
    commitment = "几乎没有"
    validation = "客户端解析"

# RGB 协议（彩色币进化）
class RGB:
    state = "完全链外"
    commitment = "Taproot commitment"
    validation = "客户端验证 + 密码学证明"
```

### ② 证明：资产可以依附在 UTXO 之上

Omni、Counterparty、RGB 都基于同一理念：资产是对 UTXO 的解释层视角。

```python
# 所有后续协议的基础抽象
class AssetProtocol:
    def attach_asset_to_utxo(self, utxo, asset):
        """
        将资产附加到 UTXO
        这是彩色币的核心洞察
        """
        pass
```

### ③ 启发了 OP_RETURN 的引入

彩色币开发者的需求直接推动了 OP_RETURN 的标准化。

```python
# 历史进程
timeline = {
    '2012': '彩色币提出，没有标准数据嵌入方式',
    '2013': 'OP_RETURN 讨论开始，彩色币是主要推动力',
    '2014': 'Bitcoin Core 0.9 正式引入 OP_RETURN (40 bytes)',
    '2015': '扩展到 80 bytes',
    '2025': '扩展到 100,000 bytes (v30)'
}
```

### ④ 确立了前提：复杂协议需要明确的数据结构

```python
# 彩色币的教训
lessons = {
    '隐式状态机': '不可靠',
    '零数据协议': '不适合复杂应用',
    '链外解释': '需要链上承诺',
    '钱包兼容性': '必须考虑生态'
}

# 后续协议的改进
improvements = {
    'Omni': '明确的 OP_RETURN 指令',
    'Counterparty': '结构化 payload',
    'RGB': '密码学承诺 + 链外状态',
    'Atomicals': 'Taproot commitment + witness'
}
```

---

## 3.9 统一框架设计

### 3.9.1 多协议引擎架构

为了对比两种协议，我们设计了一个统一的彩色币引擎：

**架构图：**

```
ColoredCoinsEngine
├── 协议选择层
│   ├── EPOBC
│   └── Open Assets
├── 状态管理层
│   ├── color_db: {(txid,vout) → ColoredCoin}
│   └── tx_db: {txid → Transaction}
├── 核心操作层
│   ├── issue_asset()    # 发行资产
│   ├── transfer_asset() # 转账资产
│   └── get_balance()    # 查询余额
└── 工具层
    ├── export_state()   # 导出状态
    └── validate()       # 验证守恒
```

**核心数据结构：**

```python
@dataclass
class ColoredCoin:
    txid: str
    vout: int
    color_id: str      # EPOBC: "txid:vout" / OA: hash(script)
    quantity: int
    protocol: ColoringProtocol
```

**关键设计模式：**

1. **策略模式**：根据协议类型选择不同的处理逻辑
2. **状态机模式**：维护 UTXO 集的状态转换
3. **工厂模式**：统一的资产创建接口

> **💡 完整实现：**  
> 完整的 500+ 行统一引擎代码、测试用例、运行示例请参考动手实践部分 Section 3。

### 3.9.2 运行示例解读

```bash
$ python ColoredCoinsEngine.py

[1] Open Assets Protocol Demo
发行 1000 个 TokenA...
  Genesis TX: genesis_3903d5b5
  Asset ID: 3903d5b563d3e8c76d62ec31bcbeb68ee1e651af

转账: 300 给 Alice, 700 给 Bob...
  Transfer TX: transfer_1

余额查询:
  Alice: {'3903d5b5...': 300}
  Bob: {'3903d5b5...': 700}
```

**技术洞察：**

1. **Asset ID 生成**：基于发行地址哈希，确保唯一性
2. **Order-Based Coloring**：输入 1000 → 输出 [300, 700]
3. **UTXO 模型**：余额 = Σ(未花费彩色UTXO)
4. **状态持久化**：JSON 格式导出完整状态



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
❌ **经济成本**：Anti-dust patch 让小额资产变得不可行

**历史地位：**

彩色币虽然失败了，但它是整个比特币资产协议历史的开端。

后续的所有协议——Omni、Counterparty、Stamps、Ordinals、Atomicals、RGB——都是在彩色币的基础上进化而来。

---

## 📚 延伸学习

### 理论深入

本章专注于彩色币的核心原理、设计思想和技术权衡。对于想要：

- **实际编写代码**
- **运行测试用例**
- **构建完整系统**
- **进行扩展实验**

请参考：**[Chapter 3 Part 2: 彩色币协议动手实践](./ch03_colored-coins_动手实践.md)**

动手实践部分包含：

1. **完整代码实现**
   - EPOBC Parser 完整实现（~100 行）
   - Open Assets Parser 完整实现（~200 行）
   - 统一引擎框架（~500 行）

2. **交互式教程**
   - nSequence 标记演示
   - LEB128 编码步骤拆解
   - Asset ID 计算详解

3. **测试套件**
   - 单元测试
   - 集成测试
   - 边界情况测试

4. **扩展实验**
   - Spent UTXO 跟踪
   - 元数据管理
   - 资产销毁机制

### 参考资源

**官方文档：**
- [Open Assets Protocol Specification](https://github.com/OpenAssets/open-assets-protocol)
- [EPOBC on Bitcoin Wiki](https://en.bitcoin.it/wiki/Colored_Coins)
- [Colored Coins Whitepaper (Meni Rosenfeld, 2012)](https://github.com/bitcoinx/colored-coins)

**开源实现：**
- [Coinprism](https://github.com/Coinprism/openassets) - Open Assets 参考实现
- [ChromaWay](https://github.com/chromaway/ngcccbase) - EPOBC 实现
- [Colu](https://github.com/Colored-Coins) - 改进版协议

---

**下一章预告：**

**第 4 章：Omni（Mastercoin）—— OP_RETURN 的第一次正式应用**

这是比特币历史上第一条"数据明确写入链上"的资产协议，也是 USDT 最初的发行平台。我们将看到彩色币的教训如何推动了 OP_RETURN 的标准化，以及 Omni 如何彻底改变了比特币资产协议的设计范式。