# Chapter 5

## Counterparty —— OP_RETURN 与裸多签的双通道协议

### 从"显式数据"走向"结构化载荷"

在 Omni 之后，人们已经接受一个事实：

**在比特币上实现资产与协议，必须显式写入数据。**

但 Omni 主要依赖单一通道：OP_RETURN。
它的载荷有限（80 bytes）、结构简单，对复杂状态机的表达能力不足。

Counterparty 则走得更远——它在工程上作了两个大胆尝试：

1. 继续使用 OP_RETURN 写协议指令
2. 利用多签公钥字段（裸多签）塞进更多结构化数据

它因此成为：

- Stamp 协议的直接先祖
- 后续 Ordinals / Atomicals "数据塞进非预期字段"的启发源
- 多通道数据嵌入的第一个成熟范本

本章的目标，是从工程角度重构 Counterparty 的设计，并解释：

- 为什么它要双通道？
- 多签里的数据究竟是怎么编码的？
- 它如何在链上实现"资产 + DEX + 协议指令"？
- 它的边界与隐患在哪里？

---

## 5.1 设计目标：不仅是资产，还要有"协议栈"

Counterparty 的野心不止于"发个代币"，而是：

- 发行资产（Token）
- 实现去中心化交易（DEX）
- 支持可编程指令（bets、contracts 等）
- 尽可能不修改比特币共识

用本书的话说：

Omni 把"状态机"搬到了 off-chain，
Counterparty 则试图在链上承载更丰富的"指令语言"。

为此，它需要两个东西：

1. 一条干净的指令通道（OP_RETURN）
2. 一条容量更大的载荷通道（multisig 公钥位）

---

## 5.2 OP_RETURN 通道：指令头 + 协议识别

和 Omni 类似，Counterparty 使用：

```
OP_RETURN <payload>
```

payload 里包含：

- 协议识别（magic）
- 指令类型（type：创建、转账、挂单……）
- 资产 ID 或 issuance 信息
- 金额 / 数量字段
- 部分附加参数

这个部分主要承担：

**"这里有一条 Counterparty 协议消息，它属于哪种类型？"**

但 Counterparty 很快就遇到问题：

**OP_RETURN 的字节数不够。**

为了承载更复杂的数据结构，它需要找到第二个出口。

---

## 5.3 裸多签通道：把数据当成"伪公钥"塞进去

多签脚本形态：

```
OP_1 <pubkey1> <pubkey2> ... <pubkeyN> OP_N OP_CHECKMULTISIG
```

每个压入栈的 `<pubkey>` 在共识层仅需满足：

- 是一个 33 字节（压缩公钥）或 65 字节（未压缩公钥）的字符串
- 椭圆曲线点合法性验证只在有限程度上检查（某些实现甚至不做）

于是 Counterparty 做出一个非常关键的工程决策：

**把任意数据编码为"伪公钥"，塞进多签脚本的公钥位置。**

### 伪公钥编码

对于压缩公钥格式（33 bytes）：

```
0x02/0x03 + 32 bytes 数据
```

其中：
- `0x02` 或 `0x03` 是压缩公钥的前缀（表示 y 坐标的奇偶性）
- 后 32 bytes 可以是任意数据

对于未压缩公钥格式（65 bytes）：

```
0x04 + 64 bytes 数据
```

其中：
- `0x04` 表示未压缩公钥
- 后 64 bytes 可以是任意数据

### 多签作为数据容器

这样的输出对 Bitcoin Core 来说，只是：

**这是一个合法的多签输出**

但对 Counterparty 解析器来说：

**这是存储协议载荷的容器（data carrier）**

它以此扩展了协议的可表达范围：

- 更大容量（每个公钥 33 bytes，可以多个公钥）
- 可以拆分到多个公钥位
- 支持复杂参数与结构化 payload

---

## 5.4 双通道协作：指令头 + 载荷体

典型模式：

- **OP_RETURN** 负责"这是什么指令"（指令类型、基本参数）
- **多签公钥位** 负责"这条指令的完整参数/数据"（扩展数据）

形象地说：

- OP_RETURN 是 HTTP 头
- 多签 payload 是 HTTP Body

### 解析流程

在解析时，Counterparty 客户端会：

1. 扫描交易中的 OP_RETURN，确认这是 Counterparty 消息
2. 再去解析同一交易中的多签输出，将公钥字段解码为数据
3. 组合成一条完整的协议指令
4. 更新链下状态机（谁拥有多少资产，谁挂了什么单）

### 示例：资产发行

```
交易结构：
  output[0]: OP_RETURN <counterparty-header>
    - magic: "CNTRPRTY"
    - type: "issuance"
    - asset_name: "MYTOKEN"
  
  output[1]: OP_1 <fake_pubkey1> <fake_pubkey2> OP_2 OP_CHECKMULTISIG
    - fake_pubkey1: 0x02 + 32 bytes (资产描述数据)
    - fake_pubkey2: 0x02 + 32 bytes (元数据扩展)
```

解析器组合后得到完整的资产发行指令。

---

## 5.5 工程复现：双通道数据嵌入

### 5.5.1 伪公钥编码函数

```python
def encode_data_as_pubkey(data: bytes) -> bytes:
    """
    将任意数据编码为伪公钥
    
    Args:
        data: 要编码的数据（最多 32 bytes）
    
    Returns:
        33 bytes 的伪压缩公钥
    """
    if len(data) > 32:
        raise ValueError("Data too long for compressed pubkey")
    
    # 填充到 32 bytes
    padded = data + b'\x00' * (32 - len(data))
    
    # 添加压缩公钥前缀（0x02 表示 y 坐标为偶数）
    return b'\x02' + padded


def decode_pubkey_as_data(pubkey: bytes) -> bytes:
    """
    从伪公钥中提取数据
    
    Args:
        pubkey: 33 bytes 的伪公钥
    
    Returns:
        原始数据（去除前缀和填充）
    """
    if len(pubkey) != 33 or pubkey[0] not in [0x02, 0x03]:
        raise ValueError("Invalid pubkey format")
    
    # 去除前缀，提取 32 bytes
    data = pubkey[1:]
    
    # 去除尾部的填充零
    return data.rstrip(b'\x00')
```

### 5.5.2 构造 Counterparty 交易

```python
def create_counterparty_tx(instruction_type: str, 
                          op_return_payload: bytes,
                          multisig_data: List[bytes]) -> dict:
    """
    创建 Counterparty 风格的双通道交易
    
    Args:
        instruction_type: 指令类型
        op_return_payload: OP_RETURN 载荷
        multisig_data: 多签载荷数据列表
    
    Returns:
        完整的交易结构
    """
    outputs = []
    
    # 1. OP_RETURN 输出（指令头）
    op_return_script = b'\x6a' + bytes([len(op_return_payload)]) + op_return_payload
    outputs.append({
        'value': 0,
        'scriptPubKey': op_return_script
    })
    
    # 2. 多签输出（载荷体）
    fake_pubkeys = [encode_data_as_pubkey(data) for data in multisig_data]
    
    multisig_script = bytearray()
    multisig_script.append(0x51)  # OP_1
    for pubkey in fake_pubkeys:
        multisig_script.append(0x21)  # OP_PUSHDATA1 (33 bytes)
        multisig_script.extend(pubkey)
    multisig_script.append(0x52)  # OP_2 (2-of-N)
    multisig_script.append(0xae)  # OP_CHECKMULTISIG
    
    outputs.append({
        'value': 600,  # dust limit
        'scriptPubKey': bytes(multisig_script)
    })
    
    return {
        'txid': 'counterparty_tx',
        'inputs': [{'txid': 'funding', 'vout': 0}],
        'outputs': outputs
    }
```

### 5.5.3 解析 Counterparty 交易

```python
def parse_counterparty_tx(tx: dict) -> dict:
    """
    解析 Counterparty 双通道交易
    
    Returns:
        解析后的协议指令
    """
    op_return_data = None
    multisig_data = []
    
    # 1. 查找 OP_RETURN
    for output in tx['outputs']:
        script = output['scriptPubKey']
        if script.startswith(b'\x6a'):  # OP_RETURN
            length = script[1]
            op_return_data = script[2:2+length]
            break
    
    # 2. 查找多签输出
    for output in tx['outputs']:
        script = output['scriptPubKey']
        if script.startswith(b'\x51'):  # OP_1
            # 解析多签脚本，提取公钥
            pos = 1
            while pos < len(script):
                if script[pos] == 0x21:  # OP_PUSHDATA1 (33 bytes)
                    pos += 1
                    pubkey = script[pos:pos+33]
                    data = decode_pubkey_as_data(pubkey)
                    multisig_data.append(data)
                    pos += 33
                elif script[pos] in [0x52, 0x53, 0xae]:  # OP_2, OP_3, OP_CHECKMULTISIG
                    break
                else:
                    pos += 1
    
    return {
        'op_return': op_return_data,
        'multisig_payload': multisig_data,
        'combined': op_return_data + b''.join(multisig_data)
    }
```

---

## 5.6 Counterparty 的协议演进

### 5.6.1 早期版本：纯 OP_RETURN

Counterparty 最初（2014年）主要使用 OP_RETURN，但很快发现容量不足。

### 5.6.2 引入多签通道

2014-2015 年，Counterparty 开始使用裸多签作为数据容器，显著扩展了协议表达能力。

### 5.6.3 SegWit 升级

2017 年 SegWit 激活后，Counterparty 也支持将数据放入 witness 中，进一步扩展了容量。

---

## 5.7 优点与代价：Counterparty 暴露的结构性问题

### 优点

- 利用现有结构（多签公钥位）扩展容量
- 不需要新 opcode
- 不改变共识
- 协议可迭代
- 表达力显著强于纯 OP_RETURN

### 代价

#### 1. 裸多签 = policy 层不喜欢的模式

- 占用 UTXO
- 难以花费（需要多个签名）
- 混淆了脚本语义（明明是数据，却伪装成密钥）
- Bitcoin Core 0.17+ 不再中继裸多签

#### 2. Block/UTXO 资源被当成数据存储

- 对节点负担较大
- 与 "UTXO should be minimal" 的理念冲突
- 永久占用 UTXO 集空间

#### 3. 解析器复杂度上升

- 必须同时解析 OP_RETURN 和 multisig
- 必须防御错误格式、攻击载荷、恶意脚本
- 需要处理多签脚本的各种变体

#### 4. 为后续"滥用脚本作为存储"树立了先例

- Stamp、部分 Ordinals 风格的尝试，直接继承了这条路
- 开启了"脚本即存储"的设计范式

---

## 5.8 Counterparty 在本书中的位置：多通道嵌入与脚本结构的第一次滥用

如果说：

- **彩色币** 是零字节协议
- **Omni** 是单通道显式数据协议

那么 **Counterparty** 就是：

**双通道嵌入 + 结构化载荷协议。**

### 它的意义有三层：

1. **证明 OP_RETURN 不够用，协议会主动寻找第二出口**
   - 当单通道容量不足时，协议设计者会寻找其他数据入口
   - 这为后续 witness、Taproot 的使用奠定了基础

2. **打开"多签字段可作为数据容器"的潘多拉魔盒**
   - 首次系统性地将脚本字段用作数据存储
   - 证明了"共识允许但语义滥用"的可行性

3. **为后续 Stamps、裸多签、脚本隐藏数据奠定实践基础**
   - Stamp 协议直接继承了 Counterparty 的多签数据容器思路
   - Ordinals 和 Atomicals 也受到这种"非预期字段利用"的启发

### 在数据嵌入史中的位置

| 协议 | 数据入口 | 特点 |
|------|---------|------|
| 彩色币 | 无（零字节） | 纯解释层 |
| Omni | OP_RETURN | 单通道，80 bytes |
| **Counterparty** | **OP_RETURN + 裸多签** | **双通道，结构化** |
| Stamp | 裸多签 | 单通道，极端化 |
| Ordinals | Witness | 单通道，大容量 |
| Atomicals | Witness + Taproot | 多通道，结构化 |

---

## 5.9 小结：Counterparty 的工程意义

Counterparty 代表：

- ✔ 多通道数据嵌入的第一个成熟范本
- ✔ 脚本结构"语义滥用"的首次系统性实践
- ✔ 证明了协议设计者会主动寻找数据入口
- ✔ 为后续协议（Stamp、Ordinals）提供了技术路径
- ✔ 暴露了裸多签作为数据容器的优缺点

并为后续发展指明方向：

**当单通道不够用时，协议会寻找第二、第三通道。**
**但每个通道都有其代价和限制。**

这就引出了下一章：

**第 6 章：Stamp —— 裸多签的极端化**

Stamp 将 Counterparty 的多签数据容器思路推向极端，完全依赖裸多签，成为"多签即存储"的典型代表。
