# Chapter 3 - Part 2

## 彩色币协议动手实践

### Hands-On Implementation of Colored Coins Protocols

本章节是《彩色币（Colored Coins）的工程复现》的动手实践部分。  
通过实际编写和运行代码，深入理解 EPOBC 和 Open Assets 两种彩色币协议的工作原理。

---

## 📋 实践概览

### 学习目标

完成本实践后，你将能够：

✅ 理解 EPOBC 的 nSequence 零开销标记机制  
✅ 实现 Open Assets 的 LEB128 编码和 Marker Output  
✅ 掌握 Order-Based Coloring 算法  
✅ 构建完整的彩色币状态管理系统  
✅ 对比两种协议的优劣

### 实践路线

```
环境准备
    ↓
实践 1: EPOBC 协议
  - nSequence 标记
  - Genesis 发行
  - Transfer 转账
    ↓
实践 2: Open Assets 协议
  - LEB128 编码
  - Marker Output
  - Asset ID 计算
    ↓
实践 3: 完整统一实现
  - 多协议支持
  - 状态管理
  - 余额查询
    ↓
测试与验证
  - 单元测试
  - 集成测试
  - 边界测试
    ↓
扩展实验
  - 混合输入检测
  - 资产销毁
  - 多资产管理
```

---

## 🔧 环境准备

### 系统要求

```bash
# Python 版本
Python 3.8+

# 必需的标准库
- hashlib  (加密哈希)
- json     (状态导出)
- dataclasses (数据结构)
- enum     (枚举类型)
- typing   (类型注解)
```

### 目录结构

```
colored-coins/
├── epobc/
│   ├── EPOBCGenesisTransaction.py
│   ├── EPOBCParser.py
│   └── test_epobc.py
├── openassets/
│   ├── OpenAssetsMarker.py
│   ├── OpenAssetsParser.py
│   └── test_open_assets.py
├── unified/
│   └── ColoredCoinsEngine.py
└── README.md
```

### 创建工作目录

```bash
mkdir -p colored-coins/{epobc,openassets,unified}
cd colored-coins
```

---

## 🎨 实践 1: EPOBC 协议实现

### 1.1 核心概念回顾

EPOBC (Enhanced Padded-Order-Based Coloring) 的关键特性：

- **零链上开销**：使用 nSequence 字段标记
- **数量耦合**：资产数量 = satoshi 数量
- **简单直接**：最早的彩色币实现思路

### 1.2 nSequence 字段标记

```python
# epobc/nsequence_demo.py

def demonstrate_nsequence_tagging():
    """
    演示 nSequence 字段的标记机制
    """
    
    # nSequence 是每个交易输入都有的 32 位字段
    # EPOBC 使用低 6 位来编码交易类型
    
    # Genesis 交易标记
    genesis_tag = 0x25  # 二进制: 100101
    print(f"Genesis Tag:")
    print(f"  十六进制: 0x{genesis_tag:02x}")
    print(f"  十进制: {genesis_tag}")
    print(f"  二进制: {bin(genesis_tag)}")
    print(f"  低6位: {genesis_tag & 0x3F}")
    
    # Transfer 交易标记
    transfer_tag = 0x33  # 二进制: 110011
    print(f"\nTransfer Tag:")
    print(f"  十六进制: 0x{transfer_tag:02x}")
    print(f"  十进制: {transfer_tag}")
    print(f"  二进制: {bin(transfer_tag)}")
    print(f"  低6位: {transfer_tag & 0x3F}")
    
    # 普通比特币交易
    normal_sequence = 0xFFFFFFFF
    print(f"\nNormal Transaction:")
    print(f"  Sequence: 0x{normal_sequence:08x}")
    print(f"  低6位: {normal_sequence & 0x3F}")
    
    # 检测逻辑
    def detect_transaction_type(sequence):
        tag = sequence & 0x3F
        if tag == 0x25:
            return "Genesis"
        elif tag == 0x33:
            return "Transfer"
        else:
            return "Regular Bitcoin"
    
    print(f"\n检测结果:")
    print(f"  0x25 → {detect_transaction_type(0x25)}")
    print(f"  0x33 → {detect_transaction_type(0x33)}")
    print(f"  0xFFFFFFFF → {detect_transaction_type(0xFFFFFFFF)}")

if __name__ == '__main__':
    demonstrate_nsequence_tagging()
```

**运行结果：**

```
Genesis Tag:
  十六进制: 0x25
  十进制: 37
  二进制: 0b100101
  低6位: 37

Transfer Tag:
  十六进制: 0x33
  十进制: 51
  二进制: 0b110011
  低6位: 51

Normal Transaction:
  Sequence: 0xffffffff
  低6位: 63

检测结果:
  0x25 → Genesis
  0x33 → Transfer
  0xffffffff → Regular Bitcoin
```

### 1.3 EPOBC 解析器实现

**完整代码见主文 3.4.2 节，这里使用该实现。**

关键代码片段解读：

```python
# epobc/EPOBCParser.py

class EPOBCParser:
    def parse_transaction(self, tx):
        # 步骤 1: 提取 nSequence 标记
        first_input = tx['inputs'][0]
        tag = first_input['sequence'] & 0x3F  # 取低6位
        
        # 步骤 2: 路由到相应处理函数
        if tag == 0x25:  # Genesis
            return self.parse_genesis(tx)
        elif tag == 0x33:  # Transfer
            return self.parse_transfer(tx)
        else:
            return None  # 普通比特币交易
```

**关键洞察：**

```python
# nSequence 标记的妙处

优点：
  ✓ 零字节开销（nSequence 本来就存在）
  ✓ 不增加交易大小
  ✓ 不需要 OP_RETURN
  ✓ 对矿工完全透明

缺点：
  ✗ 只有 6 位可用（2^6 = 64 种类型）
  ✗ 容易与其他协议冲突
  ✗ 缺乏明确的协议标识
```

### 1.4 EPOBC 完整测试

**完整测试代码见主文，这里运行并解读结果。**

```bash
cd epobc
python test_epobc.py
```

**测试输出解读：**

```
测试 5: Complete Flow (完整流程)

--- Step 1: 发行 10000 units ---
发行成功: 10000 units, 颜色ID: tx_genesis:0

# 解读：
# - Color ID = "tx_genesis:0"
# - 格式：txid:vout
# - 简单但全局唯一

--- Step 2: 转账 3000 units 给 Alice ---
转账成功:
  (tx_transfer1, 0): 3000 units
  (tx_transfer1, 1): 7000 units

# 解读：
# - Order-based coloring 按输出顺序分配
# - Output[0] 获得 3000 (Alice)
# - Output[1] 获得 7000 (找零)

--- Step 3: Alice 转账 2000 units 给 Bob ---
转账成功:
  (tx_transfer2, 0): 2000 units
  (tx_transfer2, 1): 1000 units

--- 最终状态 ---
Alice 余额: 1000 units
Bob 余额: 2000 units
Issuer 余额: 7000 units
总余额: 10000 units (应该等于 10000)

# 解读：
# ✓ 资产数量守恒
# ✓ 颜色正确传播
# ✓ 状态一致性维护
```

---

## 🔐 实践 2: Open Assets 协议实现

### 2.1 核心概念回顾

Open Assets Protocol 的关键特性：

- **显式标记**：使用 OP_RETURN marker output
- **LEB128 编码**：可变长度高效编码
- **数量解耦**：资产数量独立于 BTC 金额
- **元数据支持**：可附加 JSON 元数据

### 2.2 LEB128 编码实现与测试

```python
# openassets/leb128_tutorial.py

class LEB128Tutorial:
    """
    LEB128 (Little Endian Base 128) 编码教程
    """
    
    @staticmethod
    def encode_with_explanation(value):
        """
        带详细解释的 LEB128 编码
        """
        print(f"\n编码值: {value}")
        print(f"二进制: {bin(value)}")
        
        result = []
        iteration = 1
        
        while True:
            print(f"\n--- 迭代 {iteration} ---")
            
            # 取低 7 位
            byte = value & 0x7F
            print(f"  低7位: {bin(byte)} = 0x{byte:02x} = {byte}")
            
            # 右移 7 位
            value >>= 7
            print(f"  右移后: {value}")
            
            # 设置延续位
            if value != 0:
                byte |= 0x80
                print(f"  设置延续位: 0x{byte:02x}")
            
            result.append(byte)
            
            if value == 0:
                print(f"  完成！")
                break
            
            iteration += 1
        
        encoded = bytes(result)
        print(f"\n结果: {encoded.hex()}")
        return encoded
    
    @staticmethod
    def decode_with_explanation(data):
        """
        带详细解释的 LEB128 解码
        """
        print(f"\n解码数据: {data.hex()}")
        
        result = 0
        shift = 0
        iteration = 1
        
        for byte in data:
            print(f"\n--- 字节 {iteration}: 0x{byte:02x} ---")
            print(f"  二进制: {bin(byte)}")
            
            # 取低 7 位
            value = byte & 0x7F
            print(f"  低7位值: {value}")
            
            # 左移并累加
            result |= value << shift
            print(f"  左移 {shift} 位: {value << shift}")
            print(f"  累加结果: {result}")
            
            # 检查延续位
            has_continuation = bool(byte & 0x80)
            print(f"  延续位: {has_continuation}")
            
            if not has_continuation:
                print(f"  完成！")
                break
            
            shift += 7
            iteration += 1
        
        print(f"\n最终结果: {result}")
        return result

# 使用示例
def tutorial_main():
    tutorial = LEB128Tutorial()
    
    # 示例 1: 编码 300
    print("="*60)
    print("示例 1: 编码 300")
    print("="*60)
    encoded = tutorial.encode_with_explanation(300)
    
    print("\n" + "="*60)
    print("验证：解码回 300")
    print("="*60)
    decoded = tutorial.decode_with_explanation(encoded)
    
    assert decoded == 300, "解码失败！"
    
    # 示例 2: 编码 10000
    print("\n\n" + "="*60)
    print("示例 2: 编码 10000")
    print("="*60)
    tutorial.encode_with_explanation(10000)

if __name__ == '__main__':
    tutorial_main()
```

**运行结果：**

```
============================================================
示例 1: 编码 300
============================================================

编码值: 300
二进制: 0b100101100

--- 迭代 1 ---
  低7位: 0b101100 = 0x2c = 44
  右移后: 2
  设置延续位: 0xac

--- 迭代 2 ---
  低7位: 0b10 = 0x02 = 2
  右移后: 0
  完成！

结果: ac02

============================================================
验证：解码回 300
============================================================

解码数据: ac02

--- 字节 1: 0xac ---
  二进制: 0b10101100
  低7位值: 44
  左移 0 位: 44
  累加结果: 44
  延续位: True

--- 字节 2: 0x02 ---
  二进制: 0b10
  低7位值: 2
  左移 7 位: 256
  累加结果: 300
  延续位: False
  完成！

最终结果: 300
```

### 2.3 Marker Output 构建

**完整实现见主文 3.5.1 节。**

关键概念：

```python
# Marker Output 结构

6a                     # OP_RETURN opcode
<length>               # Payload 长度
  4f 41                # "OA" 魔数
  01 00                # Version 1
  <count>              # 资产数量个数
  <qty1> <qty2> ...    # LEB128 编码的数量列表
  <meta_len>           # 元数据长度
  <metadata>           # 可选元数据

# 实际示例 (300, 700):
6a 0a 4f 41 01 00 02 ac 02 bc 05 00
│  │  │     │     │  │     │     └─ 无元数据
│  │  │     │     │  │     └─ 700 (LEB128)
│  │  │     │     │  └─ 300 (LEB128)
│  │  │     │     └─ 2 个数量
│  │  │     └─ Version 1
│  │  └─ "OA"
│  └─ 长度 10 bytes
└─ OP_RETURN
```

### 2.4 Asset ID 计算深度解析

```python
# openassets/asset_id_demo.py

import hashlib

class AssetIDDemo:
    """
    Asset ID 计算演示
    """
    
    @staticmethod
    def compute_asset_id_step_by_step(script):
        """
        逐步展示 Asset ID 计算过程
        """
        print(f"\n原始脚本:")
        print(f"  Hex: {script.hex()}")
        print(f"  长度: {len(script)} bytes")
        
        # Step 1: SHA256
        sha256_hash = hashlib.sha256(script).digest()
        print(f"\nStep 1: SHA256")
        print(f"  结果: {sha256_hash.hex()}")
        print(f"  长度: {len(sha256_hash)} bytes (256 bits)")
        
        # Step 2: RIPEMD160
        ripemd160 = hashlib.new('ripemd160')
        ripemd160.update(sha256_hash)
        asset_id = ripemd160.digest()
        print(f"\nStep 2: RIPEMD160")
        print(f"  结果: {asset_id.hex()}")
        print(f"  长度: {len(asset_id)} bytes (160 bits)")
        
        return asset_id
    
    @staticmethod
    def demonstrate_uniqueness():
        """
        演示不同脚本产生不同 Asset ID
        """
        scripts = {
            'P2PKH Alice': b'\x76\xa9\x14' + b'alice_pubkey_hash' + b'\x88\xac',
            'P2PKH Bob': b'\x76\xa9\x14' + b'bob___pubkey_hash' + b'\x88\xac',
            'P2SH': b'\xa9\x14' + b'script___hash_val' + b'\x87',
        }
        
        print("\n" + "="*60)
        print("Asset ID 唯一性演示")
        print("="*60)
        
        asset_ids = {}
        for name, script in scripts.items():
            print(f"\n{name}:")
            asset_id = AssetIDDemo.compute_asset_id_step_by_step(script)
            asset_ids[name] = asset_id
        
        print("\n" + "="*60)
        print("唯一性验证")
        print("="*60)
        
        unique_ids = set(asset_ids.values())
        print(f"脚本数量: {len(asset_ids)}")
        print(f"唯一 Asset ID 数量: {len(unique_ids)}")
        print(f"✓ 所有 Asset ID 都不同" if len(unique_ids) == len(asset_ids) else "✗ 存在碰撞！")

if __name__ == '__main__':
    demo = AssetIDDemo()
    demo.demonstrate_uniqueness()
```

### 2.5 Open Assets 完整流程测试

**完整测试见主文，运行结果解读：**

```bash
cd openassets
python test_open_assets.py
```

**关键测试点解读：**

```
测试 4: Genesis Transaction (资产发行)

发行参数:
  发行脚本: 76a9146973737565725f7075626b65795f686173...
  发行数量: 10000 units

Genesis 交易结构:
  输出[0]: Marker Output
    数据: 6a 08 4f 41 01 00 01 90 4e 00
    解析: {'version': 1, 'asset_quantities': [10000], 'metadata': b''}
  
  输出[1]: 资产输出
    金额: 600 satoshis  ← 只需最小 dust
    脚本: 发行地址

资产状态:
  Asset ID: 6644b6e7a94ef1a8ab633f71507455243bfde817
  位置: (genesis_tx, 1)
  数量: 10000 units  ← 与 BTC 金额无关！

# 关键对比 EPOBC:
# EPOBC: 10000 units = 10000 satoshis
# Open Assets: 10000 units + 600 satoshis (最小值)
```

---

## 🎯 实践 3: 完整统一实现

### 3.1 多协议引擎设计

**完整代码见主文 3.9.1 节，这里解读核心设计。**

架构图：

```
ColoredCoinsEngine
├── Protocol Selection
│   ├── EPOBC
│   └── Open Assets
├── State Management
│   ├── color_db: {(txid,vout) → ColoredCoin}
│   └── tx_db: {txid → Transaction}
├── Core Operations
│   ├── issue_asset()
│   ├── transfer_asset()
│   └── get_balance()
└── Utilities
    ├── export_state()
    └── validate_conservation()
```

### 3.2 运行完整示例

```bash
cd unified
python ColoredCoinsEngine.py
```

**输出解读：**

```python
[1] Open Assets Protocol Demo

发行 1000 个 TokenA...
  Genesis TX: genesis_3903d5b5
  Asset ID: 3903d5b563d3e8c76d62ec31bcbeb68ee1e651af

# 技术细节：
# 1. Asset ID 取自 RIPEMD160(SHA256(script))
# 2. Genesis TXID 用前8位简化显示
# 3. Marker output 包含 [1000]

转账: 300 给 Alice, 700 给 Bob...
  Transfer TX: transfer_1

# Order-Based Coloring:
# Input:  1000 units
# Output: [300, 700]
# 映射:   Alice(300), Bob(700)

余额查询:
  Alice: {'3903d5b5...': 300}
  Bob: {'3903d5b5...': 700}

# UTXO 模型体现:
# Alice = Σ(UTXOs owned by Alice)
# Bob = Σ(UTXOs owned by Bob)

状态导出:
{
  "colored_coins": [
    {
      "txid": "genesis_3903d5b5",
      "vout": 1,
      "quantity": 1000  # ← 原始 Genesis UTXO
    },
    {
      "txid": "transfer_1",
      "vout": 1,
      "quantity": 300   # ← Alice 的 UTXO
    },
    {
      "txid": "transfer_1",
      "vout": 2,
      "quantity": 700   # ← Bob 的 UTXO
    }
  ]
}

# 注意：Genesis UTXO 应标记为 spent！
```

### 3.3 关键代码片段解读

```python
# 协议选择逻辑
def issue_asset(self, issuing_address, quantity, metadata=b''):
    if self.protocol == ColoringProtocol.OPEN_ASSETS:
        return self._issue_open_assets(...)
    elif self.protocol == ColoringProtocol.EPOBC:
        return self._issue_epobc(...)

# 设计模式：策略模式
# 优点：
# - 易于添加新协议
# - 接口统一
# - 测试独立
```

```python
# 状态管理
self.color_db: Dict[Tuple[str, int], ColoredCoin]

# Key: (txid, vout) - UTXO 标识符
# Value: ColoredCoin - 彩色币数据

# 为什么用 (txid, vout)？
# 1. 与比特币 UTXO 模型一致
# 2. 全局唯一标识
# 3. 易于查询和更新
```

---

## 🧪 实践 4: 测试与验证

### 4.1 单元测试框架

```python
# tests/test_conservation.py

import unittest
from unified.ColoredCoinsEngine import ColoredCoinsEngine, ColoringProtocol

class TestConservation(unittest.TestCase):
    """
    测试资产守恒定律
    """
    
    def test_open_assets_conservation(self):
        """Open Assets 守恒测试"""
        engine = ColoredCoinsEngine(ColoringProtocol.OPEN_ASSETS)
        
        # 发行 1000 units
        genesis_tx = engine.issue_asset('issuer', 1000)
        
        # 转账 600 + 400 = 1000
        transfer_tx = engine.transfer_asset(
            inputs=[(genesis_tx.txid, 1)],
            outputs=[('alice', 600), ('bob', 400)]
        )
        
        # 验证守恒
        alice_balance = engine.get_balance('alice')
        bob_balance = engine.get_balance('bob')
        
        total = sum(alice_balance.values()) + sum(bob_balance.values())
        self.assertEqual(total, 1000, "资产数量应守恒")
    
    def test_destruction_allowed(self):
        """测试资产可以被销毁"""
        engine = ColoredCoinsEngine(ColoringProtocol.OPEN_ASSETS)
        
        # 发行 1000 units
        genesis_tx = engine.issue_asset('issuer', 1000)
        
        # 转账只转出 500 (销毁 500)
        transfer_tx = engine.transfer_asset(
            inputs=[(genesis_tx.txid, 1)],
            outputs=[('alice', 500)]
        )
        
        # 验证销毁
        alice_balance = engine.get_balance('alice')
        total = sum(alice_balance.values())
        
        self.assertEqual(total, 500, "应该只有 500 units")
        # 500 units 被永久销毁

if __name__ == '__main__':
    unittest.main()
```

### 4.2 集成测试

```python
# tests/test_integration.py

def test_complete_lifecycle():
    """
    测试完整生命周期
    """
    engine = ColoredCoinsEngine(ColoringProtocol.OPEN_ASSETS)
    
    # 1. 发行
    print("Step 1: Genesis")
    genesis = engine.issue_asset('issuer', 10000)
    print(f"  发行: 10000 units")
    
    # 2. 第一次转账
    print("\nStep 2: Transfer to Alice")
    tx1 = engine.transfer_asset(
        inputs=[(genesis.txid, 1)],
        outputs=[('alice', 3000), ('issuer', 7000)]
    )
    print(f"  Alice: 3000")
    print(f"  Issuer: 7000")
    
    # 3. Alice 再转账
    print("\nStep 3: Alice to Bob")
    tx2 = engine.transfer_asset(
        inputs=[(tx1.txid, 1)],
        outputs=[('bob', 2000), ('alice', 1000)]
    )
    print(f"  Bob: 2000")
    print(f"  Alice: 1000")
    
    # 4. 验证最终状态
    print("\nStep 4: Final State")
    balances = {
        'Alice': sum(engine.get_balance('alice').values()),
        'Bob': sum(engine.get_balance('bob').values()),
        'Issuer': sum(engine.get_balance('issuer').values())
    }
    
    for name, balance in balances.items():
        print(f"  {name}: {balance} units")
    
    total = sum(balances.values())
    print(f"\n  总计: {total} units")
    assert total == 10000, "守恒失败！"
    print("  ✓ 资产守恒验证通过")
```

### 4.3 边界测试

```python
# tests/test_edge_cases.py

class TestEdgeCases(unittest.TestCase):
    """
    边界情况测试
    """
    
    def test_zero_transfer(self):
        """测试零数量转账"""
        engine = ColoredCoinsEngine(ColoringProtocol.OPEN_ASSETS)
        genesis = engine.issue_asset('issuer', 1000)
        
        # 尝试转账 0
        with self.assertRaises(ValueError):
            engine.transfer_asset(
                inputs=[(genesis.txid, 1)],
                outputs=[('alice', 0)]
            )
    
    def test_exceed_input(self):
        """测试输出超过输入"""
        engine = ColoredCoinsEngine(ColoringProtocol.OPEN_ASSETS)
        genesis = engine.issue_asset('issuer', 1000)
        
        # 尝试输出 2000 (超过输入 1000)
        with self.assertRaises(ValueError):
            engine.transfer_asset(
                inputs=[(genesis.txid, 1)],
                outputs=[('alice', 2000)]
            )
    
    def test_mixed_assets(self):
        """测试混合不同资产"""
        engine = ColoredCoinsEngine(ColoringProtocol.OPEN_ASSETS)
        
        # 发行两种资产
        asset_a = engine.issue_asset('issuer_a', 1000)
        asset_b = engine.issue_asset('issuer_b', 2000)
        
        # 尝试混合转账
        with self.assertRaises(ValueError):
            engine.transfer_asset(
                inputs=[
                    (asset_a.txid, 1),
                    (asset_b.txid, 1)
                ],
                outputs=[('alice', 3000)]
            )
```

---

## 🚀 实践 5: 扩展实验

### 5.1 添加 Spent 标记

```python
# extensions/spent_tracking.py

from dataclasses import dataclass

@dataclass
class ColoredCoinWithSpent:
    """
    带 spent 标记的彩色币
    """
    txid: str
    vout: int
    color_id: str
    quantity: int
    protocol: str
    spent: bool = False  # 新增

class ImprovedEngine(ColoredCoinsEngine):
    """
    改进版引擎 - 支持 spent 跟踪
    """
    
    def transfer_asset(self, inputs, outputs):
        # 标记输入为 spent
        for txid, vout in inputs:
            key = (txid, vout)
            if key in self.color_db:
                self.color_db[key].spent = True
        
        # 创建新 UTXO
        # ... (原有逻辑)
    
    def get_balance(self, address):
        """只统计未花费的 UTXO"""
        balances = {}
        
        for colored in self.color_db.values():
            if not colored.spent:  # 关键修改
                # ... 统计逻辑
        
        return balances
    
    def get_utxo_set(self):
        """获取当前 UTXO 集"""
        return {
            key: coin 
            for key, coin in self.color_db.items() 
            if not coin.spent
        }
```

### 5.2 元数据管理

```python
# extensions/metadata_demo.py

import json

class MetadataDemo:
    """
    演示元数据的使用
    """
    
    @staticmethod
    def issue_token_with_metadata():
        engine = ColoredCoinsEngine(ColoringProtocol.OPEN_ASSETS)
        
        # 创建 JSON 元数据
        metadata = {
            "name": "MyCompany Stock",
            "symbol": "MCS",
            "decimals": 2,
            "totalSupply": 1000000,
            "issuer": "MyCompany Inc.",
            "website": "https://mycompany.com"
        }
        
        # 编码为 bytes
        metadata_bytes = json.dumps(metadata).encode('utf-8')
        
        # 发行带元数据的资产
        genesis = engine.issue_asset(
            issuing_address='issuer_addr',
            quantity=1000000,
            metadata=metadata_bytes
        )
        
        print("发行了带元数据的资产:")
        print(f"  Asset ID: {genesis.txid}")
        print(f"  元数据: {metadata}")
        
        # 元数据存储在 Marker Output 中
        # 可以通过区块链浏览器查看
```

### 5.3 资产销毁机制

```python
# extensions/burn_mechanism.py

class BurnableAssets(ColoredCoinsEngine):
    """
    支持资产销毁的引擎
    """
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.burned_amount = {}  # {color_id: total_burned}
    
    def burn_asset(self, inputs, burn_amount):
        """
        明确销毁资产
        
        Args:
            inputs: 输入 UTXO
            burn_amount: 要销毁的数量
        """
        # 收集输入
        total_input = 0
        color_id = None
        
        for txid, vout in inputs:
            colored = self.color_db[(txid, vout)]
            total_input += colored.quantity
            color_id = colored.color_id
        
        # 验证销毁数量
        if burn_amount > total_input:
            raise ValueError("销毁数量超过输入")
        
        # 创建找零（如果有）
        change = total_input - burn_amount
        if change > 0:
            # 创建找零 UTXO
            pass
        
        # 记录销毁
        self.burned_amount[color_id] = \
            self.burned_amount.get(color_id, 0) + burn_amount
        
        print(f"销毁了 {burn_amount} units")
        print(f"总销毁: {self.burned_amount[color_id]} units")
```

---

## 📊 实践总结

### 对比表

| 特性 | EPOBC | Open Assets | 优劣对比 |
|------|-------|-------------|---------|
| **链上标记** | nSequence (0 bytes) | OP_RETURN (10-20 bytes) | EPOBC 更节省空间 |
| **数量表示** | = satoshis | LEB128 独立编码 | Open Assets 更灵活 |
| **元数据** | 不支持 | 支持 | Open Assets 功能更强 |
| **Asset ID** | txid:vout | hash(script) | Open Assets 更安全 |
| **实现复杂度** | 低 | 中 | EPOBC 更简单 |
| **发行成本** | 高（数量=金额） | 低（最小 dust） | Open Assets 更经济 |

### 学到的核心概念

1. **零开销标记** (EPOBC)
   - nSequence 字段的巧妙利用
   - 协议对区块链的透明性

2. **变长编码** (LEB128)
   - 空间效率优化
   - 适应不同数量范围

3. **Order-Based Coloring**
   - 按顺序映射输入到输出
   - 简单但有效的传播规则

4. **UTXO 状态管理**
   - (txid, vout) 作为唯一标识
   - 余额 = 未花费 UTXO 之和

5. **守恒定律**
   - 输出 ≤ 输入（允许销毁）
   - 不允许凭空创造

---

## 🎓 进阶挑战

### 挑战 1: 实现多签彩色币

```python
# 提示：
# - 发行地址使用 P2SH 多签脚本
# - Asset ID = RIPEMD160(SHA256(multisig_script))
# - 转账需要多个签名
```

### 挑战 2: 添加时间锁

```python
# 提示：
# - 使用 nLocktime 或 CSV
# - 彩色币只能在特定时间后转移
# - 实现"锁定期"功能
```

### 挑战 3: 构建索引器

```python
# 提示：
# - 扫描比特币区块链
# - 识别彩色币交易
# - 构建完整的 UTXO 集
# - 提供 REST API 查询
```

### 挑战 4: 与真实比特币集成

```python
# 提示：
# - 使用 bitcoin-cli 或 bitcoinlib
# - 在 testnet 发行真实彩色币
# - 使用区块链浏览器验证
# - 处理交易确认
```

---

## 📚 参考资源

### 官方文档

- [Open Assets Protocol Specification](https://github.com/OpenAssets/open-assets-protocol)
- [EPOBC on Bitcoin Wiki](https://en.bitcoin.it/wiki/Colored_Coins)
- [Colored Coins Whitepaper (2012)](https://github.com/bitcoinx/colored-coins)

### 开源实现

- [Coinprism](https://github.com/Coinprism/openassets) - Open Assets 参考实现
- [ChromaWay EPOBC](https://github.com/chromaway/ngcccbase) - EPOBC 实现
- [Colu Protocol](https://github.com/Colored-Coins) - 改进版彩色币

### 工具

- [Colorcore](https://github.com/OpenAssets/colorcore) - 命令行钱包
- [NBitcoin](https://github.com/MetacoSA/NBitcoin) - .NET Bitcoin 库（支持彩色币）

---

## ✅ 实践检查清单

完成以下所有项目后，你已经掌握了彩色币协议：

- [ ] 理解 nSequence 标记机制
- [ ] 实现 LEB128 编码/解码
- [ ] 构建 Marker Output
- [ ] 计算 Asset ID
- [ ] 实现 Order-Based Coloring
- [ ] 编写 Genesis 交易
- [ ] 编写 Transfer 交易
- [ ] 实现状态管理
- [ ] 实现余额查询
- [ ] 编写单元测试
- [ ] 处理边界情况
- [ ] 添加 Spent 跟踪
- [ ] 支持元数据
- [ ] 实现资产销毁

---

**下一步：**

完成本章实践后，继续学习 **Chapter 4: Omni（Mastercoin）**，了解第一个使用 OP_RETURN 的正式资产协议，也是 USDT 的诞生地。

彩色币为你打下了坚实的基础，后续协议都是在此基础上的演进和改进！🎨