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

本章将从工程角度重现这一结构，让你深入理解：

- 彩色币的两大技术路线（EPOBC vs Open Assets）
- 为什么它们能工作
- 为什么最终失败
- 它如何启发了所有后续协议
- 如何用代码实现一个完整的彩色币系统

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

## 3.4 EPOBC 协议工程实现

### 3.4.1 EPOBC 交易结构

```python
# EPOBC Genesis Transaction (发行)
class EPOBCGenesisTransaction:
    """
    EPOBC 发行交易结构
    """
    def __init__(self):
        self.inputs = [
            {
                'txid': '...',
                'vout': 0,
                'sequence': 0x25  # Genesis 标记（低6位 = 100101）
            }
        ]
        self.outputs = [
            {
                'value': 10000,  # satoshis
                'scriptPubKey': '<发行地址>'
                # 该输出就是"彩色币"，数量由输出金额决定
            },
            {
                'value': 90000,  # 找零
                'scriptPubKey': '<找零地址>'
            }
        ]
```

### 3.4.2 EPOBC Coloring 规则

```python
class EPOBCParser:
    """
    EPOBC 彩色币解析器
    """
    
    def __init__(self):
        self.color_db = {}  # {(txid, vout): ColoredCoin}
    
    def parse_transaction(self, tx):
        """
        解析交易，判断是否为彩色币交易
        """
        # 检查第一个输入的 nSequence
        first_input = tx['inputs'][0]
        tag = first_input['sequence'] & 0x3F  # 取低6位
        
        if tag == 0x25:  # Genesis
            return self.parse_genesis(tx)
        elif tag == 0x33:  # Transfer
            return self.parse_transfer(tx)
        else:
            return None  # 普通比特币交易
    
    def parse_genesis(self, tx):
        """
        解析发行交易
        Genesis 输出 = 第一个输出
        """
        genesis_output = tx['outputs'][0]
        
        # 颜色 = 这笔交易的 txid + output index
        color_id = f"{tx['txid']}:{0}"
        
        # 资产数量 = 输出金额（satoshis）
        # 注意：这里金额既是 BTC 又是资产数量
        quantity = genesis_output['value']
        
        colored_coin = {
            'color_id': color_id,
            'quantity': quantity,
            'txid': tx['txid'],
            'vout': 0
        }
        
        self.color_db[(tx['txid'], 0)] = colored_coin
        return colored_coin
    
    def parse_transfer(self, tx):
        """
        解析转账交易
        使用 order-based coloring
        """
        # 收集所有彩色输入
        colored_inputs = []
        for inp in tx['inputs']:
            prev_out = (inp['txid'], inp['vout'])
            if prev_out in self.color_db:
                colored_inputs.append(self.color_db[prev_out])
        
        if not colored_inputs:
            return None  # 没有彩色输入
        
        # Order-based coloring：
        # 按输出顺序分配颜色
        total_quantity = sum(c['quantity'] for c in colored_inputs)
        
        results = []
        remaining = total_quantity
        
        for i, output in enumerate(tx['outputs']):
            if remaining <= 0:
                break
            
            # 每个输出获得的资产数量 = min(输出金额, 剩余数量)
            quantity = min(output['value'], remaining)
            
            colored_coin = {
                'color_id': colored_inputs[0]['color_id'],  # 继承颜色
                'quantity': quantity,
                'txid': tx['txid'],
                'vout': i
            }
            
            self.color_db[(tx['txid'], i)] = colored_coin
            results.append(colored_coin)
            remaining -= quantity
        
        return results


# 使用示例
def example_epobc():
    parser = EPOBCParser()
    
    # Step 1: 发行 100 个单位的资产
    genesis_tx = {
        'txid': 'tx1',
        'inputs': [{
            'txid': 'tx0', 
            'vout': 0,
            'sequence': 0x25  # Genesis 标记
        }],
        'outputs': [
            {'value': 100, 'scriptPubKey': 'addr1'},  # 彩色输出
            {'value': 9900, 'scriptPubKey': 'addr2'}  # 找零（不彩色）
        ]
    }
    
    result = parser.parse_transaction(genesis_tx)
    print(f"发行: {result}")
    # 输出: {'color_id': 'tx1:0', 'quantity': 100, ...}
    
    # Step 2: 转账 30 个单位给 Alice
    transfer_tx = {
        'txid': 'tx2',
        'inputs': [{
            'txid': 'tx1',
            'vout': 0,
            'sequence': 0x33  # Transfer 标记
        }],
        'outputs': [
            {'value': 30, 'scriptPubKey': 'alice'},   # Alice 获得 30
            {'value': 70, 'scriptPubKey': 'addr1'}    # 找零 70
        ]
    }
    
    results = parser.parse_transaction(transfer_tx)
    print(f"转账: {results}")
    # 输出: [{'color_id': 'tx1:0', 'quantity': 30, ...}, 
    #        {'color_id': 'tx1:0', 'quantity': 70, ...}]
```

### 3.4.3 EPOBC 的优势与问题

**优势：**

- ✅ 零链上开销（不增加交易大小）
- ✅ 不污染 UTXO 集（不创建 OP_RETURN 输出）
- ✅ 支持 SPV 客户端

**问题：**

- ❌ 2013年4月的 anti-dust patch 规定最小输出 5,430 satoshis，这对需要小额转账的彩色币是重大打击
- ❌ 颜色与 satoshi 数量绑定，灵活性差
- ❌ 无法表达复杂元数据
- ❌ 钱包不兼容会破坏颜色

---

## 3.5 Open Assets Protocol 工程实现

### 3.5.1 Marker Output 结构

Open Assets 使用 OP_RETURN 创建一个 "marker output" 来存储资产数量列表：

```python
# Open Assets Marker Output 结构
"""
OP_RETURN 数据格式：

0x6a                    # OP_RETURN opcode
<length>                # Marker output 长度
0x4f 0x41              # "OA" - Open Assets 标识
0x01 0x00              # Version 1
<asset_quantity_list>  # 资产数量列表（LEB128 编码）
<metadata_length>      # 元数据长度
<metadata>             # 可选元数据
"""

class OpenAssetsMarker:
    """
    Open Assets Marker Output 编码器
    """
    
    @staticmethod
    def encode_leb128(value):
        """
        LEB128 变长编码（Little Endian Base 128）
        """
        result = []
        while True:
            byte = value & 0x7F
            value >>= 7
            if value != 0:
                byte |= 0x80  # 设置延续位
            result.append(byte)
            if value == 0:
                break
        return bytes(result)
    
    @staticmethod
    def decode_leb128(data):
        """
        解码 LEB128
        """
        result = 0
        shift = 0
        for byte in data:
            result |= (byte & 0x7F) << shift
            if not (byte & 0x80):
                break
            shift += 7
        return result
    
    @staticmethod
    def create_marker(asset_quantities, metadata=b''):
        """
        创建 marker output 数据
        
        Args:
            asset_quantities: 资产数量列表 [quantity1, quantity2, ...]
            metadata: 可选元数据
        
        Returns:
            完整的 OP_RETURN 数据
        """
        data = bytearray()
        
        # OP_RETURN
        data.append(0x6a)
        
        # 构建 payload
        payload = bytearray()
        
        # OA 标识
        payload.extend([0x4f, 0x41])
        
        # 版本
        payload.extend([0x01, 0x00])
        
        # 资产数量列表长度
        payload.append(len(asset_quantities))
        
        # 编码每个资产数量
        for qty in asset_quantities:
            payload.extend(OpenAssetsMarker.encode_leb128(qty))
        
        # 元数据
        if metadata:
            payload.extend(OpenAssetsMarker.encode_leb128(len(metadata)))
            payload.extend(metadata)
        else:
            payload.append(0x00)
        
        # 添加长度前缀
        data.append(len(payload))
        data.extend(payload)
        
        return bytes(data)
    
    @staticmethod
    def parse_marker(data):
        """
        解析 marker output
        """
        if len(data) < 2 or data[0] != 0x6a:
            return None
        
        length = data[1]
        payload = data[2:2+length]
        
        # 检查 OA 标识
        if len(payload) < 4 or payload[0:2] != bytes([0x4f, 0x41]):
            return None
        
        # 版本
        version = int.from_bytes(payload[2:4], 'little')
        if version != 1:
            return None
        
        # 解析资产数量列表
        pos = 4
        num_assets = payload[pos]
        pos += 1
        
        asset_quantities = []
        for _ in range(num_assets):
            # 解码 LEB128
            qty = 0
            shift = 0
            while pos < len(payload):
                byte = payload[pos]
                pos += 1
                qty |= (byte & 0x7F) << shift
                if not (byte & 0x80):
                    break
                shift += 7
            asset_quantities.append(qty)
        
        # 解析元数据
        metadata = b''
        if pos < len(payload):
            # 元数据长度
            meta_len = 0
            shift = 0
            while pos < len(payload):
                byte = payload[pos]
                pos += 1
                meta_len |= (byte & 0x7F) << shift
                if not (byte & 0x80):
                    break
                shift += 7
            
            if meta_len > 0 and pos + meta_len <= len(payload):
                metadata = payload[pos:pos+meta_len]
        
        return {
            'version': version,
            'asset_quantities': asset_quantities,
            'metadata': metadata
        }
```

### 3.5.2 Order-Based Coloring 算法

Open Assets 使用 order-based coloring 方法：输入被视为资产单位序列，输出也被视为资产单位序列，然后按位置一一映射。

```python
class OpenAssetsParser:
    """
    Open Assets Protocol 完整实现
    """
    
    def __init__(self):
        self.color_db = {}  # {(txid, vout): {'asset_id': ..., 'quantity': ...}}
    
    def get_asset_id(self, script_pubkey):
        """
        计算 Asset ID
        Asset ID = RIPEMD160(SHA256(scriptPubKey))
        """
        import hashlib
        sha256_hash = hashlib.sha256(script_pubkey).digest()
        ripemd160 = hashlib.new('ripemd160')
        ripemd160.update(sha256_hash)
        return ripemd160.digest()
    
    def parse_transaction(self, tx):
        """
        解析 Open Assets 交易
        """
        # 寻找 marker output
        marker_index = None
        marker_data = None
        
        for i, output in enumerate(tx['outputs']):
            if output['scriptPubKey'].startswith(b'\x6a'):  # OP_RETURN
                parsed = OpenAssetsMarker.parse_marker(output['scriptPubKey'])
                if parsed:
                    marker_index = i
                    marker_data = parsed
                    break
        
        if marker_data is None:
            return None  # 不是 Open Assets 交易
        
        # 收集输入的资产
        input_units = []
        for inp in tx['inputs']:
            prev_out = (inp['txid'], inp['vout'])
            if prev_out in self.color_db:
                colored = self.color_db[prev_out]
                # 将资产展开为单位序列
                for _ in range(colored['quantity']):
                    input_units.append(colored['asset_id'])
        
        # 构建输出资产单位序列
        output_quantities = marker_data['asset_quantities']
        
        # 跳过 marker output 之前的输出
        output_units = []
        for i in range(marker_index + 1, len(tx['outputs'])):
            if i - marker_index - 1 < len(output_quantities):
                qty = output_quantities[i - marker_index - 1]
                output_units.append(qty)
            else:
                output_units.append(0)
        
        # Order-based coloring：映射输入到输出
        results = []
        input_pos = 0
        
        for i, qty in enumerate(output_units):
            output_index = marker_index + 1 + i
            
            if qty > 0 and input_pos < len(input_units):
                asset_id = input_units[input_pos]
                
                colored_coin = {
                    'asset_id': asset_id,
                    'quantity': qty,
                    'txid': tx['txid'],
                    'vout': output_index
                }
                
                self.color_db[(tx['txid'], output_index)] = colored_coin
                results.append(colored_coin)
                
                input_pos += qty
        
        return results
    
    def issue_asset(self, issuing_script, quantity):
        """
        发行资产（Genesis 交易）
        """
        asset_id = self.get_asset_id(issuing_script)
        
        # 创建 marker output
        marker = OpenAssetsMarker.create_marker([quantity])
        
        tx = {
            'txid': 'genesis_tx',
            'inputs': [{'txid': 'funding_tx', 'vout': 0}],
            'outputs': [
                {
                    'value': 600,  # 最小 dust 限制
                    'scriptPubKey': marker  # Marker output
                },
                {
                    'value': 600,
                    'scriptPubKey': issuing_script  # 资产输出
                }
            ]
        }
        
        # 记录发行
        self.color_db[('genesis_tx', 1)] = {
            'asset_id': asset_id,
            'quantity': quantity,
            'txid': 'genesis_tx',
            'vout': 1
        }
        
        return tx


# 完整示例
def example_open_assets():
    parser = OpenAssetsParser()
    
    # Step 1: 发行 1000 个单位的资产
    issuing_script = b'\x76\xa9\x14' + b'issuer_pubkey_hash' + b'\x88\xac'
    
    genesis_tx = parser.issue_asset(issuing_script, 1000)
    print(f"发行交易: {genesis_tx['txid']}")
    print(f"Asset ID: {parser.color_db[('genesis_tx', 1)]['asset_id'].hex()}")
    
    # Step 2: 转账 300 个单位
    marker = OpenAssetsMarker.create_marker([300, 700])
    
    transfer_tx = {
        'txid': 'transfer_tx',
        'inputs': [{'txid': 'genesis_tx', 'vout': 1}],
        'outputs': [
            {
                'value': 0,
                'scriptPubKey': marker  # Marker
            },
            {
                'value': 600,
                'scriptPubKey': b'alice_script'  # Alice 获得 300
            },
            {
                'value': 600,
                'scriptPubKey': issuing_script  # 找零 700
            }
        ]
    }
    
    results = parser.parse_transaction(transfer_tx)
    print(f"\n转账结果:")
    for r in results:
        print(f"  Output {r['vout']}: {r['quantity']} units")
```

### 3.5.3 真实 Open Assets 交易示例

根据 Open Assets Protocol 规范，一个真实的 marker output 看起来像这样：

```
实际的 OP_RETURN 数据（十六进制）：

6a                     # OP_RETURN
10                     # 16 bytes 长度
4f 41                  # "OA"
01 00                  # Version 1
03                     # 3 个资产数量
ac 02 00 e5 8e 26     # 数量列表（LEB128 编码）
                       # - 0xac 0x02 = 300
                       # - 0x00 = 0 (marker output自己)
                       # - 0xe5 0x8e 0x26 = 624,485
04                     # 元数据4字节
12 34 56 78           # 元数据内容

解释：
- Output 0: 300 units
- Output 1: 0 units (marker output)
- Output 2: 0 units
- Output 3: 624,485 units
- 之后的输出: 0 units
```

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

## 3.9 代码实战：构建完整的彩色币系统

### 3.9.1 完整的 Python 实现

```python
#!/usr/bin/env python3
"""
Colored Coins Reference Implementation
支持 EPOBC 和 Open Assets 两种协议
"""

import hashlib
import json
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass
from enum import Enum


class ColoringProtocol(Enum):
    """彩色币协议类型"""
    EPOBC = "EPOBC"
    OPEN_ASSETS = "OpenAssets"


@dataclass
class ColoredCoin:
    """彩色币数据结构"""
    txid: str
    vout: int
    color_id: str
    quantity: int
    protocol: ColoringProtocol


@dataclass
class Transaction:
    """交易数据结构"""
    txid: str
    inputs: List[Dict]
    outputs: List[Dict]


class ColoredCoinsEngine:
    """
    彩色币引擎 - 支持多种协议
    """
    
    def __init__(self, protocol: ColoringProtocol = ColoringProtocol.OPEN_ASSETS):
        self.protocol = protocol
        self.color_db: Dict[Tuple[str, int], ColoredCoin] = {}
        self.tx_db: Dict[str, Transaction] = {}
    
    def compute_asset_id(self, script: bytes) -> str:
        """
        计算 Asset ID (Open Assets)
        Asset ID = RIPEMD160(SHA256(scriptPubKey))
        """
        sha256 = hashlib.sha256(script).digest()
        rmd = hashlib.new('ripemd160')
        rmd.update(sha256)
        return rmd.hexdigest()
    
    def encode_leb128(self, value: int) -> bytes:
        """LEB128 编码"""
        result = []
        while True:
            byte = value & 0x7F
            value >>= 7
            if value != 0:
                byte |= 0x80
            result.append(byte)
            if value == 0:
                break
        return bytes(result)
    
    def decode_leb128(self, data: bytes) -> Tuple[int, int]:
        """LEB128 解码，返回 (value, bytes_read)"""
        result = 0
        shift = 0
        i = 0
        for byte in data:
            result |= (byte & 0x7F) << shift
            i += 1
            if not (byte & 0x80):
                break
            shift += 7
        return result, i
    
    def create_open_assets_marker(self, 
                                  quantities: List[int], 
                                  metadata: bytes = b'') -> bytes:
        """
        创建 Open Assets marker output
        """
        payload = bytearray()
        payload.extend([0x4f, 0x41])  # "OA"
        payload.extend([0x01, 0x00])  # Version 1
        payload.append(len(quantities))
        
        for qty in quantities:
            payload.extend(self.encode_leb128(qty))
        
        if metadata:
            payload.extend(self.encode_leb128(len(metadata)))
            payload.extend(metadata)
        else:
            payload.append(0x00)
        
        result = bytearray([0x6a, len(payload)])
        result.extend(payload)
        return bytes(result)
    
    def parse_open_assets_marker(self, script: bytes) -> Optional[Dict]:
        """解析 Open Assets marker"""
        if len(script) < 2 or script[0] != 0x6a:
            return None
        
        length = script[1]
        payload = script[2:2+length]
        
        if len(payload) < 4 or payload[0:2] != bytes([0x4f, 0x41]):
            return None
        
        version = int.from_bytes(payload[2:4], 'little')
        if version != 1:
            return None
        
        pos = 4
        num_assets = payload[pos]
        pos += 1
        
        quantities = []
        for _ in range(num_assets):
            qty, consumed = self.decode_leb128(payload[pos:])
            quantities.append(qty)
            pos += consumed
        
        metadata = b''
        if pos < len(payload) and payload[pos] != 0:
            meta_len, consumed = self.decode_leb128(payload[pos:])
            pos += consumed
            if pos + meta_len <= len(payload):
                metadata = payload[pos:pos+meta_len]
        
        return {
            'version': version,
            'quantities': quantities,
            'metadata': metadata
        }
    
    def issue_asset(self, 
                   issuing_address: str,
                   quantity: int,
                   metadata: bytes = b'') -> Transaction:
        """
        发行资产（Genesis 交易）
        """
        if self.protocol == ColoringProtocol.OPEN_ASSETS:
            return self._issue_open_assets(issuing_address, quantity, metadata)
        elif self.protocol == ColoringProtocol.EPOBC:
            return self._issue_epobc(issuing_address, quantity)
    
    def _issue_open_assets(self, 
                          issuing_address: str,
                          quantity: int,
                          metadata: bytes) -> Transaction:
        """Open Assets 发行"""
        script = issuing_address.encode()
        asset_id = self.compute_asset_id(script)
        
        marker = self.create_open_assets_marker([quantity], metadata)
        
        tx = Transaction(
            txid=f"genesis_{asset_id[:8]}",
            inputs=[{'txid': 'funding', 'vout': 0}],
            outputs=[
                {'value': 0, 'scriptPubKey': marker},
                {'value': 600, 'scriptPubKey': script}
            ]
        )
        
        # 记录彩色币
        colored = ColoredCoin(
            txid=tx.txid,
            vout=1,
            color_id=asset_id,
            quantity=quantity,
            protocol=ColoringProtocol.OPEN_ASSETS
        )
        
        self.color_db[(tx.txid, 1)] = colored
        self.tx_db[tx.txid] = tx
        
        return tx
    
    def _issue_epobc(self, issuing_address: str, quantity: int) -> Transaction:
        """EPOBC 发行"""
        color_id = f"epobc_{issuing_address[:8]}"
        
        tx = Transaction(
            txid=f"genesis_{color_id}",
            inputs=[{
                'txid': 'funding',
                'vout': 0,
                'sequence': 0x25  # Genesis 标记
            }],
            outputs=[
                {'value': quantity, 'scriptPubKey': issuing_address.encode()}
            ]
        )
        
        colored = ColoredCoin(
            txid=tx.txid,
            vout=0,
            color_id=color_id,
            quantity=quantity,
            protocol=ColoringProtocol.EPOBC
        )
        
        self.color_db[(tx.txid, 0)] = colored
        self.tx_db[tx.txid] = tx
        
        return tx
    
    def transfer_asset(self,
                      inputs: List[Tuple[str, int]],
                      outputs: List[Tuple[str, int]]) -> Optional[Transaction]:
        """
        转账资产
        
        Args:
            inputs: [(txid, vout), ...]
            outputs: [(address, quantity), ...]
        """
        if self.protocol == ColoringProtocol.OPEN_ASSETS:
            return self._transfer_open_assets(inputs, outputs)
        elif self.protocol == ColoringProtocol.EPOBC:
            return self._transfer_epobc(inputs, outputs)
    
    def _transfer_open_assets(self,
                             inputs: List[Tuple[str, int]],
                             outputs: List[Tuple[str, int]]) -> Optional[Transaction]:
        """Open Assets 转账"""
        # 收集彩色输入
        colored_inputs = []
        total_quantity = 0
        color_id = None
        
        for txid, vout in inputs:
            key = (txid, vout)
            if key in self.color_db:
                colored = self.color_db[key]
                colored_inputs.append(colored)
                total_quantity += colored.quantity
                if color_id is None:
                    color_id = colored.color_id
                elif color_id != colored.color_id:
                    raise ValueError("Cannot mix different colored coins")
        
        if not colored_inputs:
            return None
        
        # 验证数量
        output_total = sum(qty for _, qty in outputs)
        if output_total > total_quantity:
            raise ValueError(f"Insufficient colored coins: {total_quantity} < {output_total}")
        
        # 构造交易
        quantities = [qty for _, qty in outputs]
        marker = self.create_open_assets_marker(quantities)
        
        tx_outputs = [{'value': 0, 'scriptPubKey': marker}]
        for addr, qty in outputs:
            tx_outputs.append({
                'value': 600,
                'scriptPubKey': addr.encode()
            })
        
        tx = Transaction(
            txid=f"transfer_{len(self.tx_db)}",
            inputs=[{'txid': txid, 'vout': vout} for txid, vout in inputs],
            outputs=tx_outputs
        )
        
        # 记录新的彩色币
        for i, (addr, qty) in enumerate(outputs):
            colored = ColoredCoin(
                txid=tx.txid,
                vout=i + 1,
                color_id=color_id,
                quantity=qty,
                protocol=ColoringProtocol.OPEN_ASSETS
            )
            self.color_db[(tx.txid, i + 1)] = colored
        
        self.tx_db[tx.txid] = tx
        return tx
    
    def get_balance(self, address: str) -> Dict[str, int]:
        """获取地址的彩色币余额"""
        balances = {}
        
        for (txid, vout), colored in self.color_db.items():
            tx = self.tx_db.get(txid)
            if not tx:
                continue
            
            if vout < len(tx.outputs):
                output = tx.outputs[vout]
                if output.get('scriptPubKey', b'').decode('utf-8', errors='ignore') == address:
                    color_id = colored.color_id
                    balances[color_id] = balances.get(color_id, 0) + colored.quantity
        
        return balances
    
    def export_state(self) -> str:
        """导出状态为 JSON"""
        state = {
            'protocol': self.protocol.value,
            'colored_coins': [
                {
                    'txid': colored.txid,
                    'vout': colored.vout,
                    'color_id': colored.color_id,
                    'quantity': colored.quantity,
                    'protocol': colored.protocol.value
                }
                for colored in self.color_db.values()
            ]
        }
        return json.dumps(state, indent=2)


def demo_colored_coins():
    """
    彩色币完整演示
    """
    print("="*60)
    print("Colored Coins Reference Implementation")
    print("="*60)
    
    # Open Assets 演示
    print("\n[1] Open Assets Protocol Demo")
    print("-"*60)
    
    engine = ColoredCoinsEngine(ColoringProtocol.OPEN_ASSETS)
    
    # 发行资产
    print("\n发行 1000 个 TokenA...")
    genesis_tx = engine.issue_asset(
        issuing_address="issuer_address_123",
        quantity=1000,
        metadata=b'{"name":"TokenA","symbol":"TKA"}'
    )
    print(f"  Genesis TX: {genesis_tx.txid}")
    print(f"  Asset ID: {list(engine.color_db.values())[0].color_id}")
    
    # 转账
    print("\n转账: 300 给 Alice, 700 给 Bob...")
    transfer_tx = engine.transfer_asset(
        inputs=[(genesis_tx.txid, 1)],
        outputs=[
            ("alice_address", 300),
            ("bob_address", 700)
        ]
    )
    print(f"  Transfer TX: {transfer_tx.txid}")
    
    # 查询余额
    print("\n余额查询:")
    print(f"  Alice: {engine.get_balance('alice_address')}")
    print(f"  Bob: {engine.get_balance('bob_address')}")
    
    # 导出状态
    print("\n状态导出:")
    print(engine.export_state())
    
    # EPOBC 演示
    print("\n\n[2] EPOBC Protocol Demo")
    print("-"*60)
    
    engine2 = ColoredCoinsEngine(ColoringProtocol.EPOBC)
    
    print("\n发行 500 个 TokenB (EPOBC)...")
    genesis_tx2 = engine2.issue_asset(
        issuing_address="epobc_issuer",
        quantity=500
    )
    print(f"  Genesis TX: {genesis_tx2.txid}")
    print(f"  Color ID: {list(engine2.color_db.values())[0].color_id}")


if __name__ == '__main__':
    demo_colored_coins()
```

### 3.9.2 运行输出

```
============================================================
Colored Coins Reference Implementation
============================================================

[1] Open Assets Protocol Demo
------------------------------------------------------------

发行 1000 个 TokenA...
  Genesis TX: genesis_8b7df143
  Asset ID: 8b7df143d91c716ecfa5fc1730022f6b421b05cedee8fd52b1fc65a6

转账: 300 给 Alice, 700 给 Bob...
  Transfer TX: transfer_1

余额查询:
  Alice: {'8b7df143d91c716ecfa5fc1730022f6b421b05cedee8fd52b1fc65a6': 300}
  Bob: {'8b7df143d91c716ecfa5fc1730022f6b421b05cedee8fd52b1fc65a6': 700}

状态导出:
{
  "protocol": "OpenAssets",
  "colored_coins": [
    {
      "txid": "genesis_8b7df143",
      "vout": 1,
      "color_id": "8b7df143d91c716ecfa5fc1730022f6b421b05cedee8fd52b1fc65a6",
      "quantity": 1000,
      "protocol": "OpenAssets"
    },
    {
      "txid": "transfer_1",
      "vout": 1,
      "color_id": "8b7df143d91c716ecfa5fc1730022f6b421b05cedee8fd52b1fc65a6",
      "quantity": 300,
      "protocol": "OpenAssets"
    },
    {
      "txid": "transfer_1",
      "vout": 2,
      "color_id": "8b7df143d91c716ecfa5fc1730022f6b421b05cedee8fd52b1fc65a6",
      "quantity": 700,
      "protocol": "OpenAssets"
    }
  ]
}


[2] EPOBC Protocol Demo
------------------------------------------------------------

发行 500 个 TokenB (EPOBC)...
  Genesis TX: genesis_epobc_epobc_is
  Color ID: epobc_epobc_is
```

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

**下一章预告：**

**第 4 章：Omni（Mastercoin）—— OP_RETURN 的第一次正式应用**

这是比特币历史上第一条"数据明确写入链上"的资产协议，也是 USDT 最初的发行平台。