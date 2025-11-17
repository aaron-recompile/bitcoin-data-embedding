# Chapter 4

## Omni（Mastercoin）—— OP_RETURN 的第一次正式应用

### 比特币链上数据嵌入的"第一代显式协议"

彩色币时代，人们试图在链上构建资产，但不敢写入任何数据。
它在工程上脆弱、容易被钱包逻辑破坏。
彩色币的失败让所有人意识到：

**如果不在链上写入"显式数据"，真正的状态机无法存在。**

于是，下一代协议登上舞台：
**Mastercoin（后更名为 Omni）**。

这是比特币历史上第一个明确、系统、标准化依赖链上数据的协议。
它开创了：

- 链上指令（scriptless 指令）的概念
- 链下状态机 + 链上最小承诺的架构模式
- USDT 在比特币上运行的基础
- 推动 OP_RETURN 成为比特币数据嵌入的标准通道

本章将从历史与工程两方面重建 Omni，解释它如何成为第一种"显式写数据"的资产协议。

---

## 4.0 回顾：从"数据入口地图"看 Omni 的选择

在第2章，我们建立了比特币数据嵌入的完整地图：

**Omni 的选择：**

- ✅ 使用 Output 侧的 OP_RETURN（数据在 scriptPubKey）
- ✅ 数据创建时永久可见
- ✅ 不占用 UTXO 集（provably unspendable）
- ❌ 无法享受 witness 的 75% 折扣（OP_RETURN 在 2014年，witness 在 2017年）
- ❌ 容量受限（40-80 bytes）

**为什么 Omni 选择这条路径？**

时间背景（2013-2014）：

- SegWit 尚未出现（2017年激活）
- Witness discount 不存在
- OP_RETURN 是唯一"被认可"的数据入口
- 80 bytes 对于简单指令已经足够

**Omni 在数据嵌入演进中的位置：**

```
彩色币 (2012)          → 无数据，纯解释
  ↓
Mastercoin (2013)      → 多签假公钥（UTXO 污染）
  ↓
OP_RETURN (2014)       → Bitcoin Core 回应
  ↓
Omni (2014-now)        → OP_RETURN 标准化
  ↓
Counterparty (2014)    → OP_RETURN + 多签混合
  ↓
SegWit (2017)          → Witness 出现
  ↓
Ordinals (2023)        → Witness + Taproot
```

---

## 4.1 Mastercoin 的野心：在比特币上构建"第二层金融系统"

2013 年，Mastercoin 白皮书提出一个大胆愿景：

"在比特币上构建新资产、智能合约、稳定币、去中心化交易。"

它的目标远大于彩色币，包含：

- 自定义代币发行
- 去中心化交易（DEX）
- 众筹（Crowdsale）
- 多资产层
- 合约指令

当时的以太坊还不存在，Solidity 也不存在。
**Mastercoin 是世界上第一套链上元协议（meta-protocol）**。

---

## 4.2 Mastercoin 的数据嵌入演进

**重要历史事实：Mastercoin 最初并不是使用 OP_RETURN！**

Mastercoin 的数据嵌入方式经历了三个阶段：

### 阶段1：多签公钥嵌入（2013-2014初）

**历史背景：**

- Mastercoin 白皮书发布于 **2012年1月**
- ICO 在 **2013年7月**
- 此时 OP_RETURN 尚未被 Bitcoin Core 正式支持

**技术实现：**

- 最初使用 1-of-3 multisig，将数据编码为假公钥
- 每个"公钥"实际是编码后的协议数据
- 这导致 UTXO 集污染问题

**示例：**

```
OP_1 <fake_pubkey1> <fake_pubkey2> <fake_pubkey3> OP_3 OP_CHECKMULTISIG
```

其中 `<fake_pubkey>` 实际是 Mastercoin 协议数据。

**问题：**

- ❌ 占用 UTXO 集（永久污染）
- ❌ 难以花费（需要多个签名）
- ❌ 混淆了脚本语义（明明是数据，却伪装成密钥）
- ❌ 引起 Bitcoin Core 开发者的担忧

### 阶段2：过渡期（2014年）

**Bitcoin Core 的回应：**

- **Bitcoin Core v0.9.0（2014年3月）**引入 OP_RETURN
- 初始限制 **40 bytes**
- 目的：**回应 Mastercoin 和 Counterparty 的多签数据嵌入方式**

**Bitcoin Core 开发者的考虑：**

- 与其让协议污染 UTXO 集（通过不可花费的多签输出）
- 不如提供一个正式的数据嵌入通道（OP_RETURN）

**Mastercoin 的迁移：**

- 开始逐步从多签迁移到 OP_RETURN
- 但 40 bytes 限制仍然很紧

### 阶段3：OP_RETURN 标准化（2014年后）

- **v0.11.0（2015年4月）**提升至 **80 bytes**
- Mastercoin 更名为 **Omni Layer**
- 完全迁移到 OP_RETURN + 参考输出模式

**这个历史细节的重要性：**

1. 它解释了为什么 Counterparty 也使用多签（受 Mastercoin 早期实现影响）
2. 它说明了 OP_RETURN 的引入部分是为了回应 Mastercoin 的 UTXO 污染问题
3. 这连接了第3章（彩色币）和第5章（Counterparty）的技术演进

**关键理解：**

- Mastercoin **推动了** OP_RETURN 的采用
- 但 OP_RETURN **不是** Mastercoin 的原创发明
- 而是 **Bitcoin Core 对元协议需求的回应**

---

## 4.3 Omni 的架构：链上日志 + 链下状态机

Omni 采用了一个在当时（2013）非常创新的设计：

### 架构图

```
┌─────────────────────────────────────────────┐
│           Bitcoin Blockchain                 │
│  (只存储 OP_RETURN 指令，不存储状态)          │
│                                              │
│  Block N:   [OP_RETURN: Transfer 10 USDT]  │
│  Block N+1: [OP_RETURN: Transfer 5 USDT]   │
│  Block N+2: [OP_RETURN: Issue Token X]     │
└─────────────────────────────────────────────┘
                      ↓
                      ↓ 扫描并解析
                      ↓
┌─────────────────────────────────────────────┐
│          Omni State Machine                  │
│   (维护所有资产的完整状态)                    │
│                                              │
│  Address A:  100 USDT                       │
│  Address B:  50 USDT                        │
│  Token X:    Total Supply 1000              │
└─────────────────────────────────────────────┘
```

### 关键原则

**1. 链上只是"事件日志"（Event Log）**

- 每个 OP_RETURN 是一条日志记录
- 记录"发生了什么"，而非"当前状态是什么"
- 类似于数据库的 Write-Ahead Log

**2. 链下维护"当前状态"（Current State）**

- Omni 节点从创世区块开始扫描
- 按顺序执行每一条指令
- 累积计算得出当前状态

**3. 状态一致性来自指令顺序**

- 比特币区块链保证指令的全局顺序
- 所有诚实的 Omni 节点执行相同指令序列
- 得出相同的状态结果

### 为什么这样设计？

**优势：**

- 链上数据量最小（只有指令，无状态）
- 灵活性高（可以修改状态机逻辑）
- 链上验证简单（只需验证交易有效性）

**劣势：**

- 必须扫描完整历史才能得出当前状态
- 轻客户端实现困难
- 状态机 bug 会导致共识分裂
- 无法在比特币共识层验证 Omni 状态

**这个设计深刻影响了后续协议：**

- Counterparty：相同模式
- Runes：相同模式
- BRC-20：相同模式
- RGB：改进版（client-side validation）

### 指令类型示例

Omni 客户端解析 OP_RETURN 中的指令类型：

- 指令 0x00 → Simple Send（转账）
- 指令 0x32 → Create Property（发行资产）
- 指令 0x33 → Grant Tokens（授予代币）
- 指令 0x34 → Revoke Tokens（销毁代币）
- 指令 0x50 → DEX Sell Offer（DEX 挂单）
- 指令 0x51 → DEX Accept Offer（DEX 接受订单）

---

## 4.4 Omni 的数据格式（工程解析）

Omni 的数据字段被编码为：

```
OP_RETURN <payload>
```

payload 的结构：

| 字段 | 长度 | 含义 |
|------|------|------|
| magic | 4 bytes | 协议识别（"omni" = 0x6f6d6e69） |
| version | 2 bytes | 协议版本号 |
| type | 2 bytes | 指令类型（如发行、转账、DEX 等） |
| property_id | 4 bytes | 资产 ID |
| amount | 8 bytes | 金额 |
| extra fields | 可变 | 根据类型变化 |

### 示例1：Simple Send（转账）

```
OP_RETURN 6f6d6e69000000000000001f000000003b9aca00
```

解析：

```
6f6d6e69          = "omni" (magic bytes)
0000              = version (0)
0000              = type (0 = Simple Send)
0000001f          = property_id (31 = USDT)
000000003b9aca00  = amount (1,000,000,000 = 10.00000000 USDT)
```

### 示例2：Create Property（发行资产）

```
OP_RETURN 6f6d6e690000003200000000...
```

解析：

```
6f6d6e69          = "omni" (magic bytes)
0000              = version (0)
0032              = type (0x32 = Create Property)
00000000          = property_id (0 = 新资产)
...               = 资产名称、描述等元数据
```

**Omni 协议版本演进：**

- **Omni Layer v0**：原始 Mastercoin 协议
- **Omni Layer v1**：简化后的协议，主要用于 USDT
- 当前主要使用 v1，专注于稳定币功能

---

## 4.5 完整工程复现：构建 Omni 交易

> **💡 完整代码实现：** 本节展示核心代码片段，完整可运行实现位于 `code/omni/` 目录。

### 前置知识：Omni 的两层结构

Omni 交易同时存在于两个层面：

1. **比特币层**：正常的比特币交易（传输比特币）
2. **Omni 层**：协议层的资产转移（传输 Omni 资产）

### 示例：发送 10 USDT

**第1步：构建比特币交易**

```python
# 使用 bitcoinlib 或类似工具
from bitcoinlib.transactions import Transaction

tx = Transaction()

# 添加输入
tx.add_input(sender_utxo)

# 添加 OP_RETURN 输出（Omni payload）
omni_payload = build_omni_simple_send(31, 1000000000)  # 10 USDT
tx.add_output(
    script_pubkey=create_op_return_script(omni_payload),
    value=0  # OP_RETURN 输出 value 为 0
)

# 添加接收方输出（dust）
tx.add_output(
    script_pubkey=receiver_address,
    value=546  # dust limit
)

# 添加找零输出
tx.add_output(
    script_pubkey=change_address,
    value=utxo_value - 546 - fee
)
```

**第2步：构建 Omni payload**

```python
import struct

def build_omni_simple_send(property_id, amount):
    """
    构建 Omni Simple Send 指令
    
    Args:
        property_id: 资产 ID（31 = USDT）
        amount: 金额（以最小单位计，1 USDT = 100,000,000）
    
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
    创建 OP_RETURN 脚本
    
    Args:
        payload: Omni payload bytes
    
    Returns:
        scriptPubKey bytes
    """
    script = bytearray()
    script.append(0x6a)  # OP_RETURN
    script.append(len(payload))  # 长度
    script.extend(payload)
    return bytes(script)

# 发送 10.00000000 USDT
# 1 USDT = 100,000,000 最小单位
# 10 USDT = 1,000,000,000 最小单位
payload = build_omni_simple_send(31, 1000000000)
# Result: 6f6d6e69 0000 0000 0000001f 000000003b9aca00
```

**第3步：解析 Omni 交易**

```python
def parse_omni_transaction(tx):
    """
    解析 Omni 交易
    
    Args:
        tx: 比特币交易对象
    
    Returns:
        Omni 指令字典
    """
    # 查找 OP_RETURN 输出
    for output in tx.outputs:
        script = output.script_pubkey
        if script.startswith(b'\x6a'):  # OP_RETURN
            length = script[1]
            payload = script[2:2+length]
            
            # 检查 magic bytes
            if payload[:4] != b'omni':
                continue
            
            # 解析 payload
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

**第4步：验证**

你可以使用 Omni Explorer 查看：

- https://omniexplorer.info/
- 输入交易 ID，查看 Omni 层的解析结果

**使用完整代码实现：**

上述代码片段展示了核心逻辑，完整实现（包含错误处理、类型检查等）位于 `code/omni/` 目录。

**关键理解：**

比特币节点看到的：

```
- 一笔普通交易，带一个 OP_RETURN 输出
- 转账了 546 sats 到某地址
- 其他的比特币找零
```

Omni 节点看到的：

```
- 一笔 USDT 转账
- 从地址 A 到地址 B
- 金额 10.00000000 USDT
- Property ID 31（USDT）
```

**这就是"两层协议"的本质：**

- 底层：比特币确保交易不可篡改
- 上层：Omni 解释数据的含义

---

## 4.6 OP_RETURN 是如何"改变"比特币的

Omni 的 OP_RETURN 使用非常成功，
以至于 Bitcoin Core 后来在 policy 里正式支持：

- OP_RETURN 是推荐数据入口
- 最安全、最不干扰共识
- 不占 UTXO
- 可被钱包与节点轻松识别
- 未来可随时丢弃（prunable）

从此，比特币数据嵌入的"规范入口"确立了：

**写显式数据 → 用 OP_RETURN。**

这为：

- Counterparty
- RSK
- Runes
- Taproot Assets（部分使用）
- 比特币上的所有 metadata 项目

奠定基础。

---

## 4.7 Omni 的成功与失败

### 成功点

- 开创第一代显式链上数据协议
- USDT 在比特币上成功运行多年
- 首次实现链上资产、DEX 等
- 创造了"解析器 + 状态机"架构
- 社区大规模采用了 OP_RETURN（事实标准）

### 失败点

- payload 容量有限（40 ～ 80 bytes）
- 全状态存链 → 永久膨胀
- 依赖全节点扫描，难以轻量化
- 资产逻辑复杂，状态机容易出错
- 不够表达复杂合约逻辑
- 被以太坊完全取代（通用资产功能）
- 后期被 Runes、RGB 夺走优势

最终 Omni 演变成：

**一个单一用途系统：USDT on Bitcoin。**

它的通用资产系统功能逐渐式微。

---

## 4.7.5 USDT：Omni 的最大成功（也是唯一存活的应用）

### 历史背景

- **2014年10月**，Tether 基于 Omni 发行 USDT（Property ID: 31）
- **2015年2月**，第一笔 USDT 交易上链
- 成为第一个大规模采用的比特币资产协议应用

### 技术实现

USDT 使用 Omni Simple Send（Transaction Type 0）：

```
OP_RETURN 6f6d6e69 00000000 0000001f 000000003b9aca00
          ^^^^^^   ^^^^^^^^ ^^^^^^^^ ^^^^^^^^^^^^^^^^
          "omni"   version  type=0   amount (decimal)
                            prop=31
```

**Property ID 31 的含义：**

- Property ID 是 Omni 协议中的资产标识符
- USDT = Property 31
- 所有 USDT 转账都必须包含这个 ID

### 转账过程

**1. 链上部分：**

```
Input: 发送方的比特币 UTXO
Output 0: OP_RETURN（Omni 指令）
Output 1: 接收方地址（dust amount，通常 546 sats）
Output 2: 找零地址（剩余比特币）
```

**2. 链下状态机：**

```
Omni 节点扫描区块链
  ↓
识别 OP_RETURN 中的 "omni" magic bytes
  ↓
解析 Property ID 和金额
  ↓
更新发送方和接收方的 USDT 余额
```

### 真实案例

**交易：** `21c3354ab4422faca43761826884bb4398871a2c404a2fff71be91c9502d3ba0`

这是一笔转账 10.00000000 USDT 的交易：

- **比特币层面**：转账了几千 sats
- **Omni 层面**：转账了 10 USDT
- 两个状态机并行运行，互不干扰

### 为什么 USDT 选择 Omni？

2014-2015年的选择背景：

- 以太坊尚未主网上线（2015年7月）
- 比特币是最成熟的区块链
- Omni 是当时唯一成熟的比特币资产协议
- OP_RETURN 提供了相对安全的数据嵌入方式

### 为什么 USDT 后来迁移到以太坊？

- 比特币交易费用高（2017年牛市期间）
- 以太坊 ERC-20 更灵活（智能合约）
- 链上确认时间（10分钟 vs 15秒）
- 更好的编程能力和生态系统

### 现状（2025）

- USDT on Omni 仍在运行，但交易量很小
- 大部分 USDT 在以太坊、Tron、其他链上
- Omni 协议基本成为"USDT on Bitcoin"的同义词
- 原本宏大的"第二层金融系统"愿景已消失

---

## 4.8 Omni 在比特币数据嵌入史中的地位

### 技术维度

**Omni 的三个"第一"：**

1. **第一个规模化使用 OP_RETURN 的协议**
   - 推动 Bitcoin Core 正式支持 OP_RETURN
   - 建立了"OP_RETURN = 数据通道"的范式

2. **第一个链上指令 + 链下状态机的架构**
   - 后来被 Counterparty、Runes、BRC-20 复制
   - 证明了"最小链上承诺"的可行性

3. **第一个成功的比特币资产协议**
   - USDT 运行超过10年
   - 处理了数千万笔交易

### 历史维度

技术演进时间线：

```
2012  彩色币        → 零数据，纯解释（失败）
2013  Mastercoin    → 多签嵌入（UTXO 污染）
2014  OP_RETURN     → Bitcoin Core 提供正式通道
2014  Omni          → OP_RETURN 标准化
2014  Counterparty  → 推向极限（OP_RETURN + 多签）
2017  SegWit        → Witness 出现
2021  Taproot       → Merkle 树结构
2023  Ordinals      → Witness + Taproot（新范式）
```

### 哲学维度

**Omni 证明了一个重要观点：**

"比特币不需要修改共识层，就可以承载复杂的资产系统。"

这个观点影响了整个比特币 L2 生态：

- 不要分叉比特币
- 在现有规则内创新
- 用"解释层"扩展功能

### 局限性

**Omni 也暴露了第一代协议的根本问题：**

1. **全状态扫描** → 无法轻量化
2. **无链上验证** → 依赖客户端共识
3. **容量受限** → 80 bytes 限制表达能力
4. **单一用途化** → 最终只剩 USDT

**这些问题推动了下一代协议的演进：**

- Counterparty：突破容量限制
- Stamps：追求数据不可修剪性
- Ordinals：利用 Witness discount
- RGB：完全 off-chain 状态

### 在本书结构中的位置

彩色币属于：

**第 0 代：隐式协议（不写字节）**

Omni 则属于：

**第 1 代：显式 OP_RETURN 协议（写固定长度 payload）**

在数据嵌入史中，Omni 的意义在于：

- 首次使用"正道"写数据
- 建立链上指令范式
- 把链上数据写入变得主流化
- 让资产变成比特币生态的现实
- 影响了所有后代协议（BRC-20 / Runes / RGB）

在本书的结构中，Omni 是衔接彩色币 → Counterparty / Stamp 的关键桥梁。

---

## 4.9 小结：Omni 的工程意义

Omni 代表：

- ✔ "显式数据嵌入"第一次走向规范化
- ✔ OP_RETURN 成为比特币结构性数据入口
- ✔ 状态机上链 + 指令型协议模型
- ✔ 资产协议的第一代正式实现
- ✔ USDT 能在比特币上运行的关键基础

并为后续发展指明方向：

**下一代协议要写更多数据、结构更强、表达能力更大。**

---

## 4.10 为什么需要 Counterparty？Omni 的不足

虽然 Omni 成功运行了 USDT，但它的设计有几个根本性限制：

### 1. 数据容量不足

- OP_RETURN 限制 80 bytes
- 无法表达复杂的 DEX 订单、众筹参数
- 限制了协议的表达能力

### 2. 功能单一化

- 最初的宏大愿景（DEX、合约、众筹）基本失败
- 实际只有"发行"和"转账"被广泛使用
- 复杂功能难以在 80 bytes 内实现

### 3. 无法编码复杂数据结构

- 80 bytes 难以承载结构化数据
- 无法实现链上 orderbook
- 无法编码图像、文档等

### Counterparty 的改进

- 突破 80 bytes 限制（使用多签）
- 实现了真正的 DEX
- 支持更复杂的资产类型
- 可以编码任意二进制数据

**这就引出下一章：**

**第 5 章：Counterparty —— OP_RETURN + 多签的结构化载荷**

Counterparty 如何通过"OP_RETURN + 多签"组合，
突破 Omni 的容量限制，构建更强大的资产系统？

Counterparty 直接把"嵌入数据"推向极致，甚至把数据塞进裸多签公钥里。
它是 Stamp、Ordinals 的前身。

---

**💡 本章代码实现：** 完整可运行代码位于 `code/omni/` 目录，详见该目录的 README.md。
