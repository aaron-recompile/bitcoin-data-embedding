# Chapter 2 Part 2: 实验环境

## 从裸脚本到 Commit-Reveal：测试网实战

本章将通过测试网实验，展示数据嵌入方式的演进：

1. **裸脚本/裸多签** - 数据直接暴露在 Output scriptPubKey 中
2. **P2SH/P2WSH** - Hash 承诺模式（数据在 witness 中揭示）
3. **Taproot** - Merkle 承诺模式（数据在 witness 中揭示）

我们将用实际代码展示：同样的数据，如何从"直接暴露"演变成"commit-reveal"模式。

---

## 实验环境设置

### 前置要求

```python
# 需要的 Python 库
import hashlib
import secrets
from bitcoinlib.transactions import Transaction
from bitcoinlib.scripts import Script
from bitcoinlib.keys import Key
from bitcoinlib.wallets import Wallet
import requests  # 用于与测试网节点交互
```

### 测试网配置

```python
# 测试网参数
TESTNET = True
NETWORK = 'testnet'
RPC_URL = 'https://blockstream.info/testnet/api'  # 或本地节点

# 测试数据
TEST_DATA = b"Hello, Bitcoin Data Embedding!"
```

---

## 实验 1：裸脚本（Bare Script）- 直接暴露

### 1.1 构造裸脚本输出

```python
def create_bare_script_output(data: bytes) -> dict:
    """
    创建裸脚本输出：数据直接暴露在 scriptPubKey 中
    
    结构：
    scriptPubKey: <push data> OP_DROP OP_TRUE
    """
    # 构造脚本
    script = bytearray()
    
    # Push 数据
    if len(data) <= 75:
        script.append(len(data))  # OP_PUSHDATA
    elif len(data) <= 255:
        script.append(0x4c)  # OP_PUSHDATA1
        script.append(len(data))
    else:
        script.append(0x4d)  # OP_PUSHDATA2
        script.extend(len(data).to_bytes(2, 'little'))
    
    script.extend(data)
    
    # OP_DROP 丢弃数据
    script.append(0x75)  # OP_DROP
    
    # OP_TRUE 保证脚本通过
    script.append(0x51)  # OP_TRUE
    
    return {
        'value': 600,  # dust limit
        'scriptPubKey': bytes(script),
        'type': 'bare_script',
        'data_exposed': True  # 数据直接暴露
    }
```

### 1.2 解析裸脚本输出

```python
def parse_bare_script_output(script_pubkey: bytes) -> dict:
    """
    解析裸脚本输出，提取数据
    """
    pos = 0
    data = None
    
    # 读取 push 操作
    if pos < len(script_pubkey):
        opcode = script_pubkey[pos]
        
        if opcode <= 75:  # OP_PUSHDATA
            length = opcode
            pos += 1
            data = script_pubkey[pos:pos+length]
        elif opcode == 0x4c:  # OP_PUSHDATA1
            pos += 1
            length = script_pubkey[pos]
            pos += 1
            data = script_pubkey[pos:pos+length]
        elif opcode == 0x4d:  # OP_PUSHDATA2
            pos += 1
            length = int.from_bytes(script_pubkey[pos:pos+2], 'little')
            pos += 2
            data = script_pubkey[pos:pos+length]
    
    return {
        'data': data,
        'type': 'bare_script',
        'data_exposed': True
    }
```

### 1.3 问题分析

```python
def analyze_bare_script_problems(output: dict):
    """
    分析裸脚本的问题
    """
    problems = []
    
    # 1. 数据直接暴露
    problems.append("❌ 数据直接暴露在 scriptPubKey 中，永久占用 UTXO 集")
    
    # 2. Policy 限制
    problems.append("❌ Bitcoin Core policy 可能拒绝非标准脚本")
    
    # 3. 无法享受 witness 折扣
    problems.append("❌ 数据在 base transaction 中，4 WU/byte，无折扣")
    
    # 4. UTXO 污染
    problems.append("❌ 创建不可花费的 UTXO，污染 UTXO 集")
    
    return problems
```

---

## 实验 2：裸多签（Bare Multisig）- 直接暴露

### 2.1 将数据编码为假公钥

```python
def encode_data_as_pubkey(data: bytes) -> bytes:
    """
    将数据编码为假公钥（33 bytes）
    
    格式：0x02 + 32 bytes 数据（或填充）
    """
    if len(data) > 32:
        raise ValueError("Data too long for single pubkey")
    
    # 填充到 32 bytes
    padded = data + b'\x00' * (32 - len(data))
    
    # 添加压缩公钥前缀（0x02 表示 y 坐标为偶数）
    return b'\x02' + padded


def decode_pubkey_as_data(pubkey: bytes) -> bytes:
    """
    从假公钥中提取数据
    """
    if len(pubkey) != 33 or pubkey[0] not in [0x02, 0x03]:
        raise ValueError("Invalid pubkey format")
    
    # 去除前缀，提取 32 bytes
    data = pubkey[1:]
    
    # 去除尾部的填充零
    return data.rstrip(b'\x00')
```

### 2.2 构造裸多签输出

```python
def create_bare_multisig_output(data: bytes, num_pubkeys: int = None) -> dict:
    """
    创建裸多签输出：数据编码为假公钥
    
    结构：
    scriptPubKey: OP_1 <fake_pubkey1> <fake_pubkey2> ... OP_N OP_CHECKMULTISIG
    
    如果数据超过 32 bytes，需要分割到多个公钥
    """
    # 将数据分割成 32-byte chunks
    chunk_size = 32
    chunks = [data[i:i+chunk_size] for i in range(0, len(data), chunk_size)]
    
    # 如果最后一个 chunk 不足 32 bytes，填充
    if len(chunks[-1]) < chunk_size:
        chunks[-1] = chunks[-1] + b'\x00' * (chunk_size - len(chunks[-1]))
    
    # 构造多签脚本
    script = bytearray()
    script.append(0x51)  # OP_1 (1-of-N)
    
    fake_pubkeys = []
    for chunk in chunks:
        fake_pubkey = encode_data_as_pubkey(chunk)
        fake_pubkeys.append(fake_pubkey)
        
        # Push 33-byte pubkey
        script.append(0x21)  # OP_PUSHDATA1 (33 bytes)
        script.extend(fake_pubkey)
    
    # OP_N (N-of-N)
    script.append(0x50 + len(fake_pubkeys))  # OP_N
    
    # OP_CHECKMULTISIG
    script.append(0xae)  # OP_CHECKMULTISIG
    
    return {
        'value': 600 * len(chunks),  # 每个 output 需要 dust limit
        'scriptPubKey': bytes(script),
        'type': 'bare_multisig',
        'data_exposed': True,  # 数据直接暴露（虽然编码了）
        'num_pubkeys': len(fake_pubkeys)
    }
```

### 2.3 解析裸多签输出

```python
def parse_bare_multisig_output(script_pubkey: bytes) -> dict:
    """
    解析裸多签输出，提取数据
    """
    if not script_pubkey.startswith(b'\x51'):  # OP_1
        raise ValueError("Not a 1-of-N multisig")
    
    pos = 1
    fake_pubkeys = []
    
    # 提取所有假公钥
    while pos < len(script_pubkey):
        if script_pubkey[pos] == 0x21:  # OP_PUSHDATA1 (33 bytes)
            pos += 1
            pubkey = script_pubkey[pos:pos+33]
            fake_pubkeys.append(pubkey)
            pos += 33
        elif script_pubkey[pos] >= 0x50 and script_pubkey[pos] <= 0x60:  # OP_N
            break
        else:
            pos += 1
    
    # 解码所有假公钥
    data_chunks = []
    for pubkey in fake_pubkeys:
        chunk = decode_pubkey_as_data(pubkey)
        data_chunks.append(chunk)
    
    # 合并所有 chunks
    full_data = b''.join(data_chunks).rstrip(b'\x00')
    
    return {
        'data': full_data,
        'type': 'bare_multisig',
        'data_exposed': True,
        'num_pubkeys': len(fake_pubkeys)
    }
```

---

## 实验 3：P2SH - Hash 承诺模式

### 3.1 构造 P2SH 输出（Hash 承诺）

```python
def create_p2sh_output(data: bytes) -> dict:
    """
    创建 P2SH 输出：scriptPubKey 只包含 hash 承诺
    
    结构：
    scriptPubKey: OP_HASH160 <20-byte-hash> OP_EQUAL
    
    实际数据在 redeemScript 中（解锁时在 scriptSig 中提供）
    """
    # 构造 redeemScript（包含数据）
    redeem_script = bytearray()
    
    # Push 数据
    if len(data) <= 75:
        redeem_script.append(len(data))
    else:
        redeem_script.append(0x4c)  # OP_PUSHDATA1
        redeem_script.append(len(data))
    
    redeem_script.extend(data)
    redeem_script.append(0x75)  # OP_DROP
    redeem_script.append(0x51)  # OP_TRUE
    
    redeem_script = bytes(redeem_script)
    
    # 计算 hash
    hash160 = hashlib.new('ripemd160', hashlib.sha256(redeem_script).digest()).digest()
    
    # 构造 scriptPubKey
    script_pubkey = bytearray()
    script_pubkey.append(0xa9)  # OP_HASH160
    script_pubkey.append(0x14)  # 20 bytes
    script_pubkey.extend(hash160)
    script_pubkey.append(0x87)  # OP_EQUAL
    
    return {
        'value': 600,
        'scriptPubKey': bytes(script_pubkey),
        'type': 'p2sh',
        'data_exposed': False,  # 数据不在这里！
        'redeem_script': redeem_script,  # 需要保存，解锁时用
        'hash160': hash160.hex()
    }
```

### 3.2 解锁 P2SH（揭示数据）

```python
def create_p2sh_input(redeem_script: bytes) -> dict:
    """
    创建 P2SH 输入：在 scriptSig 中提供 redeemScript（揭示数据）
    
    结构：
    scriptSig: <redeemScript>
    """
    script_sig = bytearray()
    
    # Push redeemScript
    if len(redeem_script) <= 75:
        script_sig.append(len(redeem_script))
    else:
        script_sig.append(0x4c)  # OP_PUSHDATA1
        script_sig.append(len(redeem_script))
    
    script_sig.extend(redeem_script)
    
    return {
        'scriptSig': bytes(script_sig),
        'data_revealed': True  # 数据在这里揭示
    }
```

### 3.3 关键对比

```python
def compare_bare_vs_p2sh():
    """
    对比裸脚本和 P2SH
    """
    data = b"Hello, Bitcoin!"
    
    # 裸脚本
    bare = create_bare_script_output(data)
    print("=== 裸脚本 ===")
    print(f"scriptPubKey 大小: {len(bare['scriptPubKey'])} bytes")
    print(f"数据暴露: {bare['data_exposed']}")
    print(f"数据在: Output scriptPubKey")
    print(f"成本: 4 WU/byte (base transaction)")
    
    # P2SH
    p2sh = create_p2sh_output(data)
    print("\n=== P2SH ===")
    print(f"scriptPubKey 大小: {len(p2sh['scriptPubKey'])} bytes (只有 hash!)")
    print(f"数据暴露: {p2sh['data_exposed']}")
    print(f"数据在: Input scriptSig (解锁时)")
    print(f"成本: 4 WU/byte (base transaction)")
    print(f"优势: Output 小巧，数据在解锁时揭示")
```

---

## 实验 4：P2WSH - Witness 揭示模式

### 4.1 构造 P2WSH 输出（Hash 承诺）

```python
def create_p2wsh_output(data: bytes) -> dict:
    """
    创建 P2WSH 输出：scriptPubKey 只包含 hash 承诺
    
    结构：
    scriptPubKey: OP_0 <32-byte-sha256(witnessScript)>
    
    实际数据在 witnessScript 中（解锁时在 witness 中提供）
    """
    # 构造 witnessScript（包含数据）
    witness_script = bytearray()
    
    # Push 数据
    if len(data) <= 75:
        witness_script.append(len(data))
    else:
        witness_script.append(0x4c)  # OP_PUSHDATA1
        witness_script.append(len(data))
    
    witness_script.extend(data)
    witness_script.append(0x75)  # OP_DROP
    witness_script.append(0x51)  # OP_TRUE
    
    witness_script = bytes(witness_script)
    
    # 计算 SHA256 hash
    script_hash = hashlib.sha256(witness_script).digest()
    
    # 构造 scriptPubKey
    script_pubkey = bytearray()
    script_pubkey.append(0x00)  # OP_0
    script_pubkey.append(0x20)  # 32 bytes
    script_pubkey.extend(script_hash)
    
    return {
        'value': 600,
        'scriptPubKey': bytes(script_pubkey),
        'type': 'p2wsh',
        'data_exposed': False,  # 数据不在这里！
        'witness_script': witness_script,  # 需要保存，解锁时用
        'script_hash': script_hash.hex()
    }
```

### 4.2 解锁 P2WSH（在 Witness 中揭示）

```python
def create_p2wsh_witness(witness_script: bytes) -> list:
    """
    创建 P2WSH witness：在 witness 中提供 witnessScript（揭示数据）
    
    结构：
    witness: [
        <signature>,      # 空（因为脚本是 OP_TRUE）
        <pubkey>,        # 空
        <witnessScript>  # 实际数据在这里！
    ]
    """
    return [
        b'',  # 空签名
        b'',  # 空公钥
        witness_script  # witnessScript（包含数据）
    ]
```

### 4.3 关键优势：Witness 折扣

```python
def compare_all_methods():
    """
    对比所有方法：裸脚本、P2SH、P2WSH
    """
    data = b"Hello, Bitcoin Data Embedding!" * 10  # 300 bytes
    
    methods = {
        '裸脚本': create_bare_script_output(data),
        'P2SH': create_p2sh_output(data),
        'P2WSH': create_p2wsh_output(data)
    }
    
    print("=== 数据大小对比 ===")
    for name, output in methods.items():
        print(f"\n{name}:")
        print(f"  scriptPubKey 大小: {len(output['scriptPubKey'])} bytes")
        print(f"  数据暴露: {output['data_exposed']}")
        
        if name == '裸脚本':
            print(f"  成本: {len(output['scriptPubKey']) * 4} WU (base transaction)")
        elif name == 'P2SH':
            print(f"  成本: {len(output['scriptPubKey']) * 4} WU (output)")
            print(f"        + {len(output['redeem_script']) * 4} WU (scriptSig)")
            print(f"        总计: {(len(output['scriptPubKey']) + len(output['redeem_script'])) * 4} WU")
        elif name == 'P2WSH':
            print(f"  成本: {len(output['scriptPubKey']) * 4} WU (output)")
            print(f"        + {len(output['witness_script']) * 1} WU (witness!)")
            print(f"        总计: {len(output['scriptPubKey']) * 4 + len(output['witness_script']) * 1} WU")
            print(f"  ⭐ Witness 折扣: 75% 节省!")
```

---

## 实验 5：Taproot - Merkle 承诺模式

### 5.1 构造 Taproot 输出（Merkle 承诺）

```python
def create_taproot_output(data: bytes) -> dict:
    """
    创建 Taproot 输出：scriptPubKey 只包含 tweaked pubkey（包含 Merkle root 承诺）
    
    结构：
    scriptPubKey: OP_1 <32-byte-tweaked-pubkey>
    
    实际数据在 script leaf 中（解锁时在 witness 中提供）
    """
    # 构造 script leaf（包含数据）
    script_leaf = bytearray()
    
    # Push 数据
    if len(data) <= 75:
        script_leaf.append(len(data))
    else:
        script_leaf.append(0x4c)  # OP_PUSHDATA1
        script_leaf.append(len(data))
    
    script_leaf.extend(data)
    script_leaf.append(0x75)  # OP_DROP
    script_leaf.append(0x51)  # OP_TRUE
    
    script_leaf = bytes(script_leaf)
    
    # 计算 script leaf 的 hash
    leaf_hash = hashlib.sha256(b'\xc0' + script_leaf).digest()
    
    # 简化：使用 leaf hash 作为 Merkle root（单 leaf 情况）
    merkle_root = leaf_hash
    
    # 生成内部公钥（简化示例）
    internal_pubkey = secrets.token_bytes(32)
    
    # 计算 tweaked pubkey（简化：实际需要 BIP 340 的 tweak）
    # 这里仅作演示
    tweaked_pubkey = hashlib.sha256(internal_pubkey + merkle_root).digest()[:32]
    
    # 构造 scriptPubKey
    script_pubkey = bytearray()
    script_pubkey.append(0x51)  # OP_1
    script_pubkey.append(0x20)  # 32 bytes
    script_pubkey.extend(tweaked_pubkey)
    
    return {
        'value': 600,
        'scriptPubKey': bytes(script_pubkey),
        'type': 'taproot',
        'data_exposed': False,  # 数据不在这里！
        'script_leaf': script_leaf,  # 需要保存，解锁时用
        'merkle_root': merkle_root.hex(),
        'internal_pubkey': internal_pubkey.hex()
    }
```

### 5.2 解锁 Taproot（在 Witness 中揭示）

```python
def create_taproot_witness(script_leaf: bytes, merkle_root: bytes, internal_pubkey: bytes) -> list:
    """
    创建 Taproot witness：在 witness 中提供 script leaf 和 control block（揭示数据）
    
    结构：
    witness: [
        <signature/data>,  # 空（因为脚本是 OP_TRUE）
        <script_leaf>,     # 实际数据在这里！
        <control_block>   # Merkle proof
    ]
    """
    # 构造 control block（简化示例）
    control_block = bytearray()
    control_block.append(0xc0)  # leaf version
    control_block.extend(internal_pubkey)
    # 实际还需要 Merkle path，这里简化
    
    return [
        b'',  # 空签名
        script_leaf,  # script leaf（包含数据）
        bytes(control_block)  # control block
    ]
```

### 5.3 完整对比：从裸脚本到 Taproot

```python
def evolution_comparison():
    """
    展示从裸脚本到 Taproot 的完整演进
    """
    data = b"Hello, Bitcoin Data Embedding!" * 20  # 600 bytes
    
    print("=" * 60)
    print("数据嵌入方式演进：从直接暴露到 Commit-Reveal")
    print("=" * 60)
    
    methods = {
        '1. 裸脚本': create_bare_script_output(data),
        '2. 裸多签': create_bare_multisig_output(data),
        '3. P2SH': create_p2sh_output(data),
        '4. P2WSH': create_p2wsh_output(data),
        '5. Taproot': create_taproot_output(data)
    }
    
    for name, output in methods.items():
        print(f"\n{name}:")
        print(f"  scriptPubKey 大小: {len(output['scriptPubKey'])} bytes")
        print(f"  数据位置: ", end="")
        
        if output['data_exposed']:
            print("Output scriptPubKey (直接暴露)")
            print(f"  成本: {len(output['scriptPubKey']) * 4} WU")
        else:
            print("Input witness/scriptSig (解锁时揭示)")
            if 'redeem_script' in output:
                print(f"  成本: {len(output['scriptPubKey']) * 4} WU (output)")
                print(f"        + {len(output['redeem_script']) * 4} WU (scriptSig)")
            elif 'witness_script' in output:
                print(f"  成本: {len(output['scriptPubKey']) * 4} WU (output)")
                print(f"        + {len(output['witness_script']) * 1} WU (witness) ⭐")
            elif 'script_leaf' in output:
                print(f"  成本: {len(output['scriptPubKey']) * 4} WU (output)")
                print(f"        + {len(output['script_leaf']) * 1} WU (witness) ⭐")
        
        print(f"  优势: ", end="")
        if name == '1. 裸脚本':
            print("简单直接，但污染 UTXO")
        elif name == '2. 裸多签':
            print("可扩展，但 policy 限制")
        elif name == '3. P2SH':
            print("Output 小巧，但无 witness 折扣")
        elif name == '4. P2WSH':
            print("Output 小巧 + Witness 折扣 ⭐")
        elif name == '5. Taproot':
            print("Output 小巧 + Witness 折扣 + Merkle 结构化 ⭐⭐")
```

---

## 实验 6：完整测试网示例

### 6.1 创建测试交易

```python
def create_testnet_transaction(method: str, data: bytes, funding_txid: str, funding_vout: int):
    """
    在测试网上创建实际交易
    """
    if method == 'bare_script':
        output = create_bare_script_output(data)
    elif method == 'p2wsh':
        output = create_p2wsh_output(data)
    elif method == 'taproot':
        output = create_taproot_output(data)
    else:
        raise ValueError(f"Unknown method: {method}")
    
    # 构造交易
    tx = {
        'version': 0x02000000,
        'inputs': [{
            'txid': funding_txid,
            'vout': funding_vout,
            'scriptSig': b'',
            'sequence': 0xffffffff
        }],
        'outputs': [output],
        'locktime': 0
    }
    
    return tx, output
```

### 6.2 解析链上交易

```python
def parse_testnet_transaction(txid: str):
    """
    从测试网解析交易，识别数据嵌入方式
    """
    # 从 API 获取交易
    response = requests.get(f"{RPC_URL}/tx/{txid}/hex")
    tx_hex = response.json()
    
    # 解析交易
    # ... 解析逻辑 ...
    
    # 识别数据嵌入方式
    for output in outputs:
        script_pubkey = output['scriptPubKey']
        
        # 检查是否是 OP_RETURN
        if script_pubkey[0] == 0x6a:  # OP_RETURN
            print("类型: OP_RETURN")
            data = script_pubkey[2:]  # 跳过 OP_RETURN 和长度
            print(f"数据: {data}")
        
        # 检查是否是 P2WSH
        elif script_pubkey[0] == 0x00 and script_pubkey[1] == 0x20:
            print("类型: P2WSH")
            print("数据在 witness 中（需要解析 witness）")
        
        # 检查是否是 Taproot
        elif script_pubkey[0] == 0x51 and script_pubkey[1] == 0x20:
            print("类型: Taproot (P2TR)")
            print("数据在 witness 中（需要解析 witness）")
```

---

## 总结：演进路径

```
直接暴露模式（2.3 节）
├── 裸脚本: <push data> OP_DROP OP_TRUE
│   └── 问题: 污染 UTXO，无折扣
│
└── 裸多签: OP_1 <fake_pubkey>... OP_CHECKMULTISIG
    └── 问题: Policy 限制，无折扣

    ↓ 演进

Commit-Reveal 模式（2.4 节）
├── P2SH: OP_HASH160 <hash> OP_EQUAL
│   └── 数据在: Input scriptSig
│   └── 优势: Output 小巧
│   └── 问题: 无 witness 折扣
│
├── P2WSH: OP_0 <sha256> 
│   └── 数据在: Input witness
│   └── 优势: Output 小巧 + Witness 折扣 ⭐
│
└── Taproot: OP_1 <tweaked_pubkey>
    └── 数据在: Input witness (script leaf)
    └── 优势: Output 小巧 + Witness 折扣 + Merkle 结构化 ⭐⭐
```

**关键洞察：**

1. **从直接暴露到 Commit-Reveal**：数据从 Output 移到 Input（解锁时揭示）
2. **从 Base 到 Witness**：数据从 base transaction 移到 witness（享受折扣）
3. **从 Hash 到 Merkle**：承诺从单一 hash 到 Merkle 树（结构化）

这就是比特币数据嵌入的演进史！

---

## 下一步

- 在测试网上实际部署这些交易
- 对比不同方法的实际成本
- 分析 policy 层对不同方法的接受度
- 探索更大数据量的处理方式

**实验目标：** 通过实际代码理解"为什么现代协议选择 P2WSH/Taproot 而不是裸脚本"。

