# Chapter 7

## P2WSH —— 脚本变成"合法的数据容器"

### 从滥用结构到设计内生的容器

在 Counterparty 和 Stamps 的多签实验之后，比特币进入了 SegWit 时代。
SegWit 带来了一个关键结构：

**P2WSH（Pay-to-Witness-Script-Hash）**

它表面上是 P2SH 的 witness 化版本，
实质上却第一次在协议层给出了一个"结构化脚本容器"。

如果说：

- **Stamps** 是"偷偷往脚本里塞数据"

那么 **P2WSH** 则是：

**公开地承认脚本本身可以承载复杂结构，在 witness 里完整暴露。**

这一章要解释：

- P2WSH 在结构上到底做了什么改变
- 为什么它天然适合作为"数据容器"
- 它与 OP_RETURN / 多签相比有什么不同
- 它如何为 Ordinals、Atomicals 做铺垫

---

## 7.1 从 P2SH 到 P2WSH：脚本哈希与脚本暴露

### P2SH（Pay-to-Script-Hash）

回顾 P2SH：

```
scriptPubKey: OP_HASH160 <20-byte-hash> OP_EQUAL
```

脚本本体（redeemScript）在花费时才暴露，在 scriptSig 中提供。

**特点：**
- 脚本哈希：RIPEMD160(SHA256(redeemScript)) = 20 bytes
- 脚本在 scriptSig 中（影响 TXID）
- 数据在 base transaction 中（4 weight units per byte）

### P2WSH（Pay-to-Witness-Script-Hash）

SegWit 引入 P2WSH：

```
scriptPubKey: OP_0 <32-byte sha256(witnessScript)>
witness:      <... witnessScript and stack ...>
```

**关键变化：**

1. **脚本从 scriptSig 移到 witness 区域**
   - 不再影响 TXID
   - 解决 malleability 问题

2. **脚本哈希改用 sha256，长度 32 字节**
   - 更安全的哈希算法
   - 更大的哈希空间

3. **witness 是独立计费区间（weight），对 block 容量更宽容**
   - Base transaction: 4 weight units per byte
   - Witness data: 1 weight unit per byte
   - **75% 的成本折扣**

从数据嵌入视角看：

**witnessScript = 一个可以容纳任意复杂结构的字段。**

---

## 7.2 为什么 P2WSH 是"干净的容器"，而 Stamps 是"滥用"？

核心差异在于"语义"：

- **Stamps** 把本应是公钥的字段变成 data
  - 语义错位：公钥应该是椭圆曲线点，却被当作数据
  - 混淆了脚本的语义

- **P2WSH** 则把本来就是脚本的字段用作复杂逻辑表达
  - 语义正确：脚本本身就是用来表达逻辑的
  - 符合设计意图

### 脚本的本质

脚本本质就是：

**一个可执行的验证程序。**

因此，用脚本表达复杂逻辑、状态承诺，本就是它的原生作用；
只是在 P2WSH + witness 的组合下，这种作用第一次得到了：

- 更大的容量（witness 可以很大）
- 更清晰的结构（witness 分离）
- 更合理的费用模型（weight discount）
- 更好的工程可控性（不影响 TXID）

---

## 7.3 P2WSH 作为"协议容器"的典型模式

在 P2WSH 模式下，一个自定义协议可以这样构造：

### 1. 使用 witnessScript 表达逻辑结构

```
witnessScript:
  <protocol_magic> <version> <payload_hash> OP_DROP
  OP_TRUE
```

其中：
- `<protocol_magic>`：用于识别是哪个协议
- `<version>`：版本号
- `<payload_hash>`：链下大数据的哈希（比如图片、JSON、状态机）
- `OP_DROP`：丢弃与验证无关部分
- `OP_TRUE`：最小逻辑保证脚本返回 true

### 2. 使用 witness 传递运行时参数

```
witness: [
  <signature>,
  <pubkey>,
  <witnessScript>,  # 完整的脚本
  <additional_payload>  # 可选：额外数据
]
```

### 3. 使用 sha256(witnessScript) 作为 scriptPubKey 的承诺点

```
scriptPubKey: OP_0 <32-byte sha256(witnessScript)>
```

**作用：**
- 保证脚本不可被修改
- 为链下状态提供 anchor
- 允许选择性 reveal

从"嵌入数据"的视角看：

**witnessScript + witness 一起构成了一个"结构化数据包"。**

---

## 7.4 工程示例：用 P2WSH 承载协议数据

### 7.4.1 构造 P2WSH 数据容器

```python
import hashlib

def create_p2wsh_data_container(protocol_magic: bytes,
                                version: int,
                                payload_hash: bytes,
                                additional_data: bytes = b'') -> dict:
    """
    创建 P2WSH 数据容器
    
    Args:
        protocol_magic: 协议标识（如 "MYPROTO"）
        version: 协议版本
        payload_hash: 链下数据的哈希
        additional_data: 额外的链上数据
    
    Returns:
        完整的 P2WSH 交易结构
    """
    # 构造 witnessScript
    witness_script = bytearray()
    
    # 推送协议标识
    witness_script.append(len(protocol_magic))
    witness_script.extend(protocol_magic)
    
    # 推送版本号
    witness_script.append(version)
    
    # 推送 payload hash
    witness_script.append(len(payload_hash))
    witness_script.extend(payload_hash)
    
    # 推送额外数据（如果有）
    if additional_data:
        witness_script.append(len(additional_data))
        witness_script.extend(additional_data)
    
    # OP_DROP 丢弃所有数据
    witness_script.append(0x75)  # OP_DROP
    witness_script.append(0x75)  # OP_DROP
    witness_script.append(0x75)  # OP_DROP
    if additional_data:
        witness_script.append(0x75)  # OP_DROP
    
    # OP_TRUE 保证脚本通过
    witness_script.append(0x51)  # OP_TRUE
    
    witness_script = bytes(witness_script)
    
    # 计算脚本哈希
    script_hash = hashlib.sha256(witness_script).digest()
    
    # 构造 scriptPubKey
    script_pubkey = bytearray()
    script_pubkey.append(0x00)  # OP_0
    script_pubkey.append(0x20)  # OP_PUSHDATA1 (32 bytes)
    script_pubkey.extend(script_hash)
    
    # 构造交易
    tx = {
        'txid': 'p2wsh_tx',
        'inputs': [{'txid': 'funding', 'vout': 0}],
        'outputs': [
            {
                'value': 600,  # dust limit
                'scriptPubKey': bytes(script_pubkey)
            }
        ]
    }
    
    # witness（在花费时提供）
    witness = [
        b'',  # 空签名（因为脚本是 OP_TRUE）
        b'',  # 空公钥
        witness_script  # 完整的 witnessScript
    ]
    
    return {
        'tx': tx,
        'witness': witness,
        'witness_script': witness_script,
        'script_hash': script_hash.hex()
    }
```

### 7.4.2 解析 P2WSH 数据容器

```python
def parse_p2wsh_data_container(script_pubkey: bytes, 
                               witness: list) -> dict:
    """
    解析 P2WSH 数据容器
    
    Args:
        script_pubkey: P2WSH 的 scriptPubKey
        witness: witness 数据
    
    Returns:
        解析后的协议数据
    """
    # 验证 scriptPubKey 格式
    if not (script_pubkey[0] == 0x00 and script_pubkey[1] == 0x20):
        raise ValueError("Not a P2WSH scriptPubKey")
    
    script_hash = script_pubkey[2:34]
    
    # 从 witness 中提取 witnessScript
    witness_script = witness[-1]  # 最后一个元素通常是脚本
    
    # 验证哈希
    computed_hash = hashlib.sha256(witness_script).digest()
    if computed_hash != script_hash:
        raise ValueError("Witness script hash mismatch")
    
    # 解析 witnessScript
    pos = 0
    protocol_magic = None
    version = None
    payload_hash = None
    additional_data = None
    
    # 读取协议标识
    if pos < len(witness_script):
        magic_len = witness_script[pos]
        pos += 1
        protocol_magic = witness_script[pos:pos+magic_len]
        pos += magic_len
    
    # 读取版本号
    if pos < len(witness_script):
        version = witness_script[pos]
        pos += 1
    
    # 读取 payload hash
    if pos < len(witness_script):
        hash_len = witness_script[pos]
        pos += 1
        payload_hash = witness_script[pos:pos+hash_len]
        pos += hash_len
    
    # 读取额外数据（如果有）
    if pos < len(witness_script):
        # 跳过 OP_DROP 和 OP_TRUE
        # 实际实现需要更复杂的解析
        pass
    
    return {
        'protocol_magic': protocol_magic,
        'version': version,
        'payload_hash': payload_hash.hex() if payload_hash else None,
        'additional_data': additional_data
    }
```

### 7.4.3 完整示例

```python
def p2wsh_demo():
    """P2WSH 数据容器演示"""
    
    # 创建数据容器
    result = create_p2wsh_data_container(
        protocol_magic=b"MYPROTO",
        version=1,
        payload_hash=hashlib.sha256(b"off_chain_data").digest(),
        additional_data=b"extra_info"
    )
    
    print("P2WSH 数据容器:")
    print(f"  Script Hash: {result['script_hash']}")
    print(f"  Witness Script 长度: {len(result['witness_script'])} bytes")
    
    # 解析
    parsed = parse_p2wsh_data_container(
        result['tx']['outputs'][0]['scriptPubKey'],
        result['witness']
    )
    
    print("\n解析结果:")
    print(f"  协议: {parsed['protocol_magic']}")
    print(f"  版本: {parsed['version']}")
    print(f"  Payload Hash: {parsed['payload_hash']}")
```

---

## 7.5 与 OP_RETURN、裸多签的对比

| 方式 | 数据位置 | 语义合理性 | 容量 | Weight成本 | 典型用途 |
|------|---------|----------|------|-----------|---------|
| OP_RETURN | Output | 高 | 80 bytes (v29) / 100KB (v30+) | 4 WU/byte | 指令、短 payload |
| 裸多签 | Output (pubkey) | 低 | 中（每公钥 33 bytes） | 4 WU/byte | Stamps 等极端例子 |
| P2WSH | **Input witness** | **高** | **极大** | **1 WU/byte** | **结构化协议、脚本应用** |

**总结：**

- **OP_RETURN** 适合"短指令"
- **P2WSH** 适合"结构化逻辑与承诺"

P2WSH 的出现，使得"脚本作为数据结构"的路线从滥用变成设计内生的可能性。

---

## 7.6 P2WSH 的数据位置：承诺-揭示模式

**关键洞察：**

P2WSH 遵循"承诺-揭示"模式：

- **Output 侧**：只存储脚本哈希（32 bytes）
  ```
  scriptPubKey: OP_0 <32-byte sha256(witnessScript)>
  ```
  - 小巧、不占空间
  - 提供密码学承诺

- **Input 侧**：在 witness 中揭示完整脚本
  ```
  witness: [<signature>, <pubkey>, <witnessScript>]
  ```
  - 数据实际在这里
  - 享受 75% 的成本折扣

这与第二章中讨论的"承诺-揭示模式"完全一致。

---

## 7.7 P2WSH 为后续协议铺路

### 7.7.1 为 Ordinals 铺路

Ordinals 使用 witness 承载图片数据，本质上就是 P2WSH 思路的延伸：

- 数据在 witness 中（享受折扣）
- 使用 Taproot script path（更结构化）
- 不污染 UTXO 集（witness 可剪枝）

### 7.7.2 为 Atomicals 铺路

Atomicals 使用 Taproot Merkle 树存储数据，也是 P2WSH 承诺模式的升级：

- 链上承诺（Taproot commitment）
- 链下数据（witness 中的 script leaf）
- 选择性 reveal

### 7.7.3 为 RGB 铺路

RGB 的客户端验证模式也受到 P2WSH 的启发：

- 链上承诺（最小数据）
- 链下状态（完整数据）
- 客户端验证（密码学证明）

---

## 7.8 P2WSH 在本书中的位置：连接脚本时代与 Taproot 时代的桥梁

在"比特币非交易数据史"的叙事里，P2WSH 是一个过渡节点：

- 它没有直接催生某个爆款协议（不像 Ordinals / RGB）
- 但它提供了一个更健康的"嵌入数据 + 脚本逻辑"的结构
- 为 Taproot script path、Ordinals witness 模式做好铺垫
- 让"脚本作为容器"从实践走向规范

### 在数据嵌入史中的位置

| 协议 | 数据位置 | 特点 | 时代 |
|------|---------|------|------|
| Counterparty | Output (multisig) | 双通道，占用 UTXO | Legacy |
| Stamp | Output (multisig) | 极端化，UTXO 污染 | Legacy |
| **P2WSH** | **Input witness** | **结构化，享受折扣** | **SegWit Era** |
| Ordinals | Input witness | 大容量，图片存储 | Taproot Era |
| Atomicals | Input witness + Taproot | Merkle 化，结构化 | Taproot Era |

---

## 7.9 小结：P2WSH 的工程意义

P2WSH 代表：

- ✔ 脚本作为数据容器的"合法化"
- ✔ 承诺-揭示模式的首次成熟应用
- ✔ Witness discount 的经济优势
- ✔ 为后续协议（Ordinals、Atomicals、RGB）奠定基础
- ✔ 从"滥用结构"到"设计内生"的转折点

并为后续发展指明方向：

**数据应该放在 witness 中，享受成本折扣。**
**脚本可以作为结构化数据容器。**

这就引出了下一章：

**第 8 章：Ordinals —— 用 witness 把 sat 变成"载体"**

witness 时代的数据爆炸，Ordinals 将 P2WSH 的思路推向极致，直接在 witness 中存储图片和内容。
