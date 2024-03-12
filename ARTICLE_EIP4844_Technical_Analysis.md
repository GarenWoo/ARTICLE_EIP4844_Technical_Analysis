# EIP-4844 背景与技术解读

**前言**：EIP-4844 是以太坊 Dencun 升级中最重要的提案，标志着以太坊在以去中心化方式扩容的道路上迈出了切实而重要的一步。随着 Dencun 升级于 2024.3.13 上线以太坊主网，EIP-4844 也将得到实施。

## A. 概述

**EIP-4844** 引入新的数据存储类型“blob”（Binary Large Object），是以太坊网络一种新的用于存储和传输的数据存储类型，旨在为 Layer 2 扩容方案提供数据可用性保障，且保证较低的数据存储和传输的成本。从而实现临时的扩容解决方案并为未来的**完全分片**提供基础。

引入新的交易格式“**blob-carrying transactions**”，其中包含的 blob 数据，EVM 无需访问 blob 数据本身，仅通过其**承诺**（commitment）即可验证 blob 数据的正确性和完整性。从而实现临时的扩容解决方案并为未来的**完全分片**提供基础。

该 EIP 可理解是**完全分片**彻底实施前的“先行版本”。



## B. 背景

目前 Rollups 显着降低了许多以太坊用户的费用，但费用依然对于许多用户来说是昂贵的。Rollup 的相对昂贵源自于 Calldata 有限的存储空间和 Layer2 数据对 Layer1 较高的存储需求。Layer2 的交易数据并不需要永久存储于昂贵的以太坊 Layer1 上，显然 Calldata 不适宜作为 Layer2 交易数据的存储空间。从链上数据可知，目前使用 Layer2 的成本多数来自于 Calldata 数据永久存储成本。

<figure>   <img src="https://img.learnblockchain.cn/pics/20240312194453.png" alt="Layer2 使用费用">   <figcaption>Layer2 使用费用（数据来自 l2fees.info）</figcaption> </figure>

<br>

<figure>   <img src="https://img.learnblockchain.cn/pics/20240312194518.png" alt="Layer2(OP)交易成本多数来自 Layer1 数据存储">   <figcaption>Layer2(OP)交易成本多数来自 Layer1 数据存储（数据来自 Dune.com）</figcaption> </figure>

解决 **Rollup** 本身长期不足的长期解决方案一直是**数据分片**，这将可用 **Rollups** 的链添加约 16 MB 的专用数据空间。然而，**数据分片**仍需要相当长的时间才能完成实施和部署。

以太坊即将进行的 **Dencun 升级**属于以太坊升级路线图 “**The Surge**” 的一部分，旨在通过 **Proto-Danksharding** 的引⼊，进⼀步提⾼以太坊的交易吞吐量和可扩展性。

EIP-4844 被用于提高可扩展性和效率，扩容方案被称为 "**Proto-Danksharding**"，名称来自该扩容思路的以太坊研究员 Dankrad Feist 和 Proto Lambda。其中，“**Danksharding**” 是一种[分片解决方案](https://ethereum.org/en/roadmap/danksharding/)，通过将以太坊网络分割成多个“片段”以分散工作负载并提高交易处理能力。前缀 "**Proto-**"表示它是该分片解决方案的初步或初始阶段，旨在显著提高以太坊的可扩展性。

<figure>   <img src="https://img.learnblockchain.cn/pics/20240312194540.jpg" alt="Proto-DankshardingWorkingProcess">   <figcaption>Proto-Dankshanrding 工作原理</figcaption> </figure>

此 EIP 通过实现将在**完全分片**中使用的**交易格式**，先行解决数据存储和传输的效率问题，为将来分片带来的数据管理挑战提前做准备，从而提供了一个权宜之计的解决方案。此 EIP 的改进和数据结构能够无缝集成，避免了未来可能需要的大规模重构。

与**完全分片**相比，该 **EIP** 压低了可包含的 blob 交易数据存储的指标，相当于每块约 0.375 MB 的目标容量大小和约 0.75 MB 的容量上限。



## C. 技术细节

### 1. 数据存储格式 blob

新引入的数据存储格式 blob 本质是一个字节向量（`ByteVector[n]`），`n = FIELD_ELEMENTS_PER_BLOB * BYTES_PER_FIELD_ELEMENT`。

- `FIELD_ELEMENTS_PER_BLOB`：新增常量，表示每个 **blob** 中的字段数，为固定值 4096 。
- `BYTES_PER_FIELD_ELEMENT`：新增常量，表示每一个 **blob 字段**的存储字节数，为固定值 32 。

一个 **blob** 的可用容量为 4096 * 32 个字节，约为 **0.125 MB** 。

此 EIP 还新增了部分常量：

- `MIN_EPOCHS_FOR_BLOB_SIDECARS_REQUESTS`：固定值 4096，定义了节点必须存储 blob 数据的最小时段数，以便在必要时可以检索这些数据（4096 * 32 * 12 / 3600 / 24 = ~18.2 天）

**blob 数据的组成部分**：

1. **用户数据**：这是 blob 的核心内容，即要在以太坊上存储和传输的实际数据。对于不同的应用，这些数据可能包括但不限于交易详情、状态变更信息或任何需要在区块链上存储的大量数据。
2. **数据承诺**：为了确保数据的完整性和可验证性，blob 通常会包含一个数据承诺，使得任何人都可以验证 blob 中数据的正确性，而无需访问整个数据集。

---

### 2. blob 交易

**blob 交易**是一种符合 **[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)** 交易框架的交易类型。

[**EIP-2718**](https://eips.ethereum.org/EIPS/eip-2718) 定义了一种交易类型的框架，使得后续的 EIP 所引入新的交易类型可以向后兼容：

    1. `TransactionType || TransactionPayload` 为有效的交易。
    
    2. `TransactionType || ReceiptPayload` 为有效的交易收据。

其中，`TransactionType` 定义交易格式，`TransactionPayload` 和 `ReceiptPayload` 分别是交易和收据内容。**EIP-4844** 所引入的**blob 交易**向前兼容此交易类型的框架。


- **blob 交易**的 `TransactionType` 是常量 `BLOB_TX_TYPE`，其值为`Bytes1(0x03)`。

- **blob 交易**的 `TransactionPayload` 是如下 `TransactionPayloadBody` 的 **RLP 序列化结果**。

  ```python
  TransactionPayloadBody = [chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, to, value, data, access_list, max_fee_per_blob_gas, blob_versioned_hashes, y_parity, r, s]
  
  TransactionPayload = rlp(TransactionPayloadBody)
  ```

  <aside>
  💡 RLP（递归长度前缀，Recursive Length Prefix）是以太坊中用于编码任意数据结构的主要序列化方法。
  </aside>


  - **blob 交易 `TransactionPayloadBody` 各字段解释如下**：

    - **blob 交易常规字段**（与 **EIP-1559** 的语义相同）：

      - `chain_id`：链 ID 。
      - `nonce`：发送者账户的交易计数器。
      - `max_priority_fee_per_gas`：最大优先费用（小费）。
      - `max_fee_per_gas`：最大总费用（包括基础费用）每单位 gas 。
      - `gas_limit`：交易可使用的最大gas 量。
      - `value`：以 wei 为单位的发送的以太币数量。
      - `data`：交易的输入数据。
      - `access_list`：一个访问列表，包括交易执行过程中需要访问的**地址**和**存储键**，以优化 gas 消耗和提高交易执行效率。

    - **blob 交易非常规字段**（与 **EIP-1559** 的语义不同）：

      - `to` ：接收方的**20 字节地址**，不同于 **EIP-1559**，但它不能为 `nil`。这意味着 **blob 交易**不能用来创建合约。

    - **blob 交易独有字段**：

      - `max_fee_per_blob_gas` ：`uint256`，用户指定的愿意给出的每单位 **blob gas** 的最大费用。
      - `blob_versioned_hashes`：方法 {**kzg_to_versioned_hash**} 的哈希输出列表，即一个数组，一个区块用到几个 blob 数组内就会有几个元素，每个元素为一个 blob 的 `VersionedHash` （即**版本化哈希值**，对 blob 的数据进行承诺，由方法 {**kzg_to_versioned_hash**} 基于 **KZG 承诺**生成）。

- **blob 交易**的 `ReceiptPayload`（在 **[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)** 中定义） 是 `TransactionPayloadBody`  的 **RLP 序列化结果**。

  ```python
  TransactionPayloadBody = [status, cumulative_transaction_gas_used, logs_bloom, logs]
  
  ReceiptPayload = rlp(TransactionPayloadBody)
  ```


  - **blob 交易 `TransactionPayloadBody` 各字段解释如下**：

    - `status`：交易执行的状态码（成功或失败）。
    - `cumulative_transaction_gas_used`：执行到当前交易为止累计的 gas 消耗量。
    - `logs_bloom`：交易产生的日志的Bloom过滤器，用于快速检索日志事件。
    - `logs`：交易执行过程中产生的日志事件列表。


---

### 3. blob 交易的签名

blob 交易的发送者对交易数据进行的数字签名，先将 `BLOB_TX_TYPE` 与 `TransactionPayload` 的拼接结果作为消息原文做 Keccak256 哈希处理，获得**摘要**如下：

`keccak256(BLOB_TX_TYPE || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, to, value, data, access_list, max_fee_per_blob_gas, blob_versioned_hashes]))`

再将**摘要**进行 ***secp256k1 签名*** 获得签名值 `y_parity`、`r` 和 `s`。

---

### 4. blob gas 设计

引入 **blob gas** 作为一种新型 gas。它独立于普通 gas 并遵循自己的目标规则（类似于 **EIP-1559**）。

1. **单个 blob 的 gas 容量**由新增常量 `GAS_PER_BLOB` 定义，值为 `2 ** 17`，即 131072 单位个 gas。
2. **目标 blob gas 消耗量**由新增常量 `TARGET_BLOB_GAS_PER_BLOCK` 定义，对应于 3 个 blob 的 gas 容量，即 `393216` 单位个 gas（3 * `GAS_PER_BLOB`）。
3. **最大 blob gas 消耗量**由新增常量`MAX_BLOB_GAS_PER_BLOCK` 定义，对应于 6 个 blob 的 gas 容量，即 `786432` 单位个 gas（6 * `GAS_PER_BLOB`）。

#### blob 基础费更新规则

**blob 基础费更新规则遵循下列公式**： 

`blob_base_fee = MIN_BLOB_BASE_FEE * e**(excess_blob_gas / BLOB_BASE_FEE_UPDATE_FRACTION)`

- `MIN_BLOB_BASE_FEE`：此 **EIP** 新增常量，表示最小的 blob gas 单价，为固定值 1 wei。
- `excess_blob_gas` 是“从此 EIP 实施后至今各区块 **blob gas** 的消耗量”相比“**目标 blob gas 数量**”多出的 **blob gas** 数量的加总，是整个网络全局的累积值。与 **EIP-1559** 类似，它是一个自我修正公式：随着超额费用增加，`blob_base_fee` 呈**指数**增长，减少使用量并最终迫使超额费用回落。
- `BLOB_BASE_FEE_UPDATE_FRACTION`是新增的常量，值为 3338477 ，是 blob 基础费更新的分数值，用于调整交易费用。
- 指数计算通过方法 {**fake_exponential**} 来近似实现。

例如：当区块 `N` 消耗 `X` **blob gas** 时，区块 `N+1` 中的 `excess_blob_gas` 会增加 `X - TARGET_BLOB_GAS_PER_BLOCK`，因此区块 `N+1` 的 `blob_base_fee` 会增加 `e**((X - TARGET_BLOB_GAS_PER_BLOCK) / BLOB_BASE_FEE_UPDATE_FRACTION)` 倍。因此，它与现有的 **EIP-1559** 具有类似的效果，但更“稳定”，因为无论它如何分布，它都会以相同的方式响应相同的总使用量。

参数 `BLOB_BASE_FEE_UPDATE_FRACTION` 控制 **blob** 基本费用的最大变化率，将每个块的最大变化率 `e**(TARGET_BLOB_GAS_PER_BLOCK / BLOB_BASE_FEE_UPDATE_FRACTION)` ≈ 1.125 作为目标。

**区块头字段** `excess_blob_gas` （即 `header.excess_blob_gas`）来存储“计算  **blob gas** 基本费用所需”的持久数据。目前，只有 **blob** 以 **blob gas** 定价。

#### blob gas 计算

```python
# 计算一笔交易的 blob 基础费总值
def calc_data_fee(header: Header, tx: Transaction) -> int:
    return get_total_blob_gas(tx) * get_blob_base_fee(header)

# 计算一笔交易的 blob gas 消耗总值
def get_total_blob_gas(tx: Transaction) -> int:
    return GAS_PER_BLOB * len(tx.blob_versioned_hashes)

# 计算一个区块的 blob 基础费单价
def get_blob_base_fee(header: Header) -> int:
    return fake_exponential(
        MIN_BLOB_BASE_FEE,
        header.excess_blob_gas,
        BLOB_BASE_FEE_UPDATE_FRACTION
    )
```

除此之外， 共识层还将增加对 **blob gas** 的区块有效性检查。

**blob 基础费**在交易执行前**从发送方余额中扣除并销毁，交易失败时不予退还**。

---

### 5. 区块头扩展

此 **EIP** 实施后，**区块头**将被加入 2 个新字段（64 位无符号整型）：

- `blob_gas_used` 是区块内交易消耗的 **blob gas** 总量（即处理 **blob 数据**所需的 gas 量）。
- `excess_blob_gas` 是在出块之前，**累计**超出目标 **blob gas** 消耗的总量。
  - 当一个区块的 `blob_gas_used` 高于目标 **blob gas** 消耗的区块会增加 `excess_blob_gas` 的值；
  - 当一个区块的 `blob_gas_used` 低于目标 **blob gas**  消耗的区块会降低 `excess_blob_gas` 的值（以 0 为界）。

因此，**区块头 RLP 编码**也将包含这 2 个新字段：

```python
rlp([
    parent_hash,
    0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347, # ommers hash
    coinbase,
    state_root,
    txs_root,
    receipts_root,
    logs_bloom,
    0, # difficulty
    number,
    gas_limit,
    gas_used,
    timestamp,
    extradata,
    prev_randao,
    0x0000000000000000, # nonce
    base_fee_per_gas,
    withdrawals_root,
    blob_gas_used,
    excess_blob_gas,
])
```

`excess_blob_gas` 的值可以使用**父区块头**计算，它与 blob 基础费的计算密切相关（详见后文“blob 基础费更新规则”）。

```python
def calc_excess_blob_gas(parent: Header) -> int:
    if parent.excess_blob_gas + parent.blob_gas_used < TARGET_BLOB_GAS_PER_BLOCK:
        return 0
    else:
        return parent.excess_blob_gas + parent.blob_gas_used - TARGET_BLOB_GAS_PER_BLOCK
```

**如何理解 `excess_blob_gas` 的算法：**
根据定义，`excess_blob_gas` 是一个**累积量**，因此其计算公式可以理解为：
当前区块的 `excess_blob_gas` = `parent.excess_blob_gas + (parent.blob_gas_used - TARGET_BLOB_GAS_PER_BLOCK)`
**括号内的部分**即为 父区块的 blob gas 消耗量相比目标 **blob gas** 的**偏差**。父区块与这个**偏差**相加，获得当前区块的 `excess_blob_gas`，即最新的 **blob gas** 消耗偏差的**累积量**。

1. 这个**偏差**为正时，累积量 `excess_blob_gas` 会进一步增加，表示网络中最新区块对应的父区块的 **blob gas** 消耗量依然高于目标消耗量；
2. 这个**偏差**为负时，累积量 `excess_blob_gas` 会降低（但是不低于 0），表示网络中最新区块对应的父区块的 **blob gas** 消耗量低于目标消耗量。

对于分叉后的第一个区块，`parent.blob_gas_used` 和 `parent.excess_blob_gas` 都被设定为 `0` 。

-------

### 6. 获取版本化哈希值的操作码

**与 blob 数据哈希计算相关的新增常量**：

- `HASH_OPCODE_BYTE`：对 blob 数据哈希计算的操作码，为固定值 `bytes1(0x49)`。

- `HASH_OPCODE_GAS`：blob 数据哈希计算所对应的 gas 消耗量，为固定值 3 。

此 EIP 添加一条指令 `BLOBHASH`（操作码为 `HASH_OPCODE_BYTE`），该指令从**堆栈**顶部读取 `index` 为**大端序**（big-endian）格式的 `uint256`，如果 `index < len(tx.blob_versioned_hashes)` ，则在堆栈上用 `tx.blob_versioned_hashes[index]` 替换它，否则栈顶被替换为全零的 `bytes32` 值。该操作码的 gas 成本为 `HASH_OPCODE_GAS`。

注：**大端序**（big-endian）是一种字节存储的排序方式。最高有效字节写在地址最低位，而最低有效字节写在地址的最高位。越靠左的字节越位于高位，而地址索引号越小则表示地址越低。

---

### 7. 点评估预编译

***KZG proof*** 是用于验证一个数据集对应的 KZG 承诺 的正确性的证明（这确保了数据的接收方可以验证数据的发送方实际上承诺了特定的数据集，而无需接收整个数据集本身）。这个 ***KZG Proof*** 声称 **blob**（由**承诺**表示）在**给定点（given point）**评估为**给定值（given value）**，即某个多项式 **`p(x)`**（由一个**承诺**表示）在给定点 **`z`** 上的值 **`p(z)`** 等于 **`y`** 。

通过在固定地址为 `POINT_EVALUATION_PRECOMPILE_ADDRESS` （常量，值为 `Bytes20(0x0A)`）预编译合约，实现对 ***KZG Proof*** 的验证 。

预编译的 gas 消耗为 `POINT_EVALUATION_PRECOMPILE_GAS` （本 EIP 新增常量，为固定值 50000）。

执行**点评估预编译**的方法 {**point_evaluation_precompile**} 做了 2 件事：

- 验证：“由给定的 **KZG 承诺** 所计算出的**版本化哈希**（通过新增的 {**kzg_to_versioned_hash**} 方法计算）”是否等于“给定的**版本化哈希**”。
- 验证：给定的 ***KZG Proof*** 是否有效（通过 [verify_kzg_proof()](https://github.com/ethereum/consensus-specs/blob/86fb82b221474cc89387fa6436806507b3849d88/specs/deneb/polynomial-commitments.md#verify_kzg_proof) 计算）。

方法 {**point_evaluation_precompile**} 的唯一参数 input 是一个 bytes 变量，其实际长度应为 192 ，是包含多个参数拼接起来的字节序列，包含如下参数：

- `versioned_hash`（bytes32）：这个 **blob** 的**版本化哈希**
- `z`（bytes32）：**给定点（given point）**
- `y`（bytes32）：**给定值（given value）**
- `commitment`（bytes48）：**KZG 承诺**，作为入参在方法{**kzg_to_versioned_hash**} 中计算出一个**版本化哈希**，并于给定的**版本化哈希** `versioned_hash` 的值做对比。
- `proof`（bytes48）：***KZG Proof***

```python
def point_evaluation_precompile(input: Bytes) -> Bytes:
    """
    Verify p(z) = y given commitment that corresponds to the polynomial p(x) and a KZG proof.
    Also verify that the provided commitment matches the provided versioned_hash.
    """
    # The data is encoded as follows: versioned_hash | z | y | commitment | proof | with z and y being padded 32 byte big endian values（检查输入的字节序列长度是否为 192 位）
    assert len(input) == 192
    # 从输入中相应截取 versioned_hash, z, y, commitment, proof
    versioned_hash = input[:32]
    z = input[32:64]
    y = input[64:96]
    commitment = input[96:144]
    proof = input[144:192]

    # Verify commitment matches versioned_hash（验证：提供的承诺计算出的版本化哈希与提供的版本化哈希是否相等）
    assert kzg_to_versioned_hash(commitment) == versioned_hash

    # Verify KZG proof with z and y in big endian format（验证：KZG proof 是否有效）
    assert verify_kzg_proof(commitment, z, y, proof)

    # Return FIELD_ELEMENTS_PER_BLOB and BLS_MODULUS as padded 32 byte big endian values
    return Bytes(U256(FIELD_ELEMENTS_PER_BLOB).to_be_bytes32() + U256(BLS_MODULUS).to_be_bytes32())
```

返回 **`FIELD_ELEMENTS_PER_BLOB`** 和 **`BLS_MODULUS`**，这两个值被填充为 bytes32 **大端序**的格式。

**预编译**必须拒绝**非规范字段元素**（即**提供的字段元素必须严格小于** **`BLS_MODULUS`**）。

1. `BLS_MODULUS`：本 EIP 新增常量，用于 **BLS**（Boneh-Lynn-Shacham）签名体系的**有限字段**的**模数**（modulus），为固定值 `52435875175126190479447740508185965837690552500527637822603658699938581184513` 。
2. **有限字段**：有限字段的大小通常由一个主要的质数或某个质数的幂来定义，这个数称为字段的**模数**。在椭圆曲线密码学中，这个**模数**通常用来定义椭圆曲线上点的坐标所属的数学范围。

---

### 8. 共识层验证

在*共识层*上，**blob** **数据**被引用，但不是编码在**信标区块体**中，而是被作为“***sidecar***”单独传播。

术语“sidecars”（侧车）的解释：泛指与主区块链交易数据相关联但不直接存储在区块链上的附加信息，并非 blob 一个单独的组成部分，包括但不限于签名、证明、元数据等。

本 EIP 中的“侧车”设计允许 **blob** 数据与**信标区块体**分开传播，使 **blob** **数据**可以独立地被网络中的节点接收和处理。这种“***sidecar***”设计，将 `is_data_available()` 黑盒化，为进一步的数据增加提供了**前向兼容性**：通过**完全分片**，`is_data_available()` 可以被*数据可用性采样 (DAS, data-availability-sampling)* 取代，从而避免所有 **blob** 被所有**信标节点**下载。

**数据可用性采样** (DAS, data-availability-sampling)：Danksharding 提出的一个方案，用于实现降低节点负担的同时也保证了数据可用性。其思想是将 blob 中的数据切割成数据碎片，并且让节点由下载 blob 数据转变为随机抽查 blob 数据碎片，让 blob 的数据碎片分散在以太坊的每个节点中，但是完整的 blob 数据却保存在整个以太坊账本中，前提是节点需要足够多且去中心化。

***共识层*** 具体的更改内容（**[ethereum/consensus-specs](https://github.com/ethereum/consensus-specs)** 代码库也定义了更改内容）：

- **信标链**：更新**信标区块**的处理流程，确保 **blob** 可用。
- **P2P 网络**：在网络中传播和同步更新后的**信标区块**；
- **诚实验证者**：生成包含 **blob 数据**的**信标链区块**；签署并发布关联的 **blob *sidecar***。

---

### 9. 执行层验证

在*执行层*，区块的有效性验证方法 {validate_block} 新增如下逻辑：

```python
def validate_block(block: Block) -> None:
    ...

    # check that the excess blob gas was updated correctly
    assert block.header.excess_blob_gas == calc_excess_blob_gas(block.parent.header)
		
		# 将 blob gas 消耗总量初始化为 0
    blob_gas_used = 0

    for tx in block.transactions:
        ...

        # modify the check for sufficient balance
        # 计算传统 gas 总费用的最大值
        max_total_fee = tx.gas * tx.max_fee_per_gas

        # 当检测到 tx 类型为 blob 交易时：
        if get_tx_type(tx) == BLOB_TX_TYPE:
          
        # 为 gas 总费用额外添加 blob 的费用最大值，得到收取的 gas 总费用最大值
            max_total_fee += get_total_blob_gas(tx) * tx.max_fee_per_blob_gas
          
        # 断言用户余额充足
        assert signer(tx).balance >= max_total_fee

        ...

        # add validity logic specific to blob txs（当检测到 tx 类型为 blob 交易时：）
        if get_tx_type(tx) == BLOB_TX_TYPE:
            # there must be at least one blob（要求至少有一个 blob 存在）
            assert len(tx.blob_versioned_hashes) > 0

            # all versioned blob hashes must start with VERSIONED_HASH_VERSION_KZG（要求所有 blob 的 versionedHash 都是以常量`VERSIONED_HASH_VERSION_KZG`开头）
            for h in tx.blob_versioned_hashes:
                assert h[0] == VERSIONED_HASH_VERSION_KZG

            # ensure that the user was willing to at least pay the current blob base fee（用户愿意支付的最大 blob gas 单价应当小于当前 blob 基础费单价）
            assert tx.max_fee_per_blob_gas >= get_blob_base_fee(block.header)

            # keep track of total blob gas spent in the block（更新 blob gas 消耗总量）
            blob_gas_used += get_total_blob_gas(tx)

    # ensure the total blob gas spent is at most equal to the limit（断言 blob gas 消耗总量小于单区块的 blob gas 数量上限）
    assert blob_gas_used <= MAX_BLOB_GAS_PER_BLOCK

    # ensure blob_gas_used matches header（断言 blob gas 消耗总量等于区块头记录的 blob gas 消耗总量）
    assert block.header.blob_gas_used == blob_gas_used
```

---

### 10. 网络

**blob 交易**的网络传播有 2 种表示形式：`PooledTransactions` 和`BlockBodies`。

- 第 1 种：`PooledTransactions`（**交易 gossip 传播**），**blob 交易**的 **EIP-2718** `TransactionPayload` 被包装为：

  ```python
  rlp([tx_payload_body, blobs, commitments, proofs])
  ```

  每个元素的定义如下：

  - `tx_payload_body` -  **[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)** 标准 blob 交易的 `TransactionPayloadBody`
  - `blobs`：`blob` 项目列表
  - `commitments`：对应于  `blobs` 的 `KZGCommitment` 列表
  - `proofs`：对应于 `blobs` 和 `commitments` 的 `KZGProof` 列表

  节点必须验证 `tx_payload_body` 并根据它验证包装的数据。应确保：

  - 有相同数量的 `tx_payload_body`.`blob_versioned_hashes`、`blobs`、`commitments` 和 `proofs` （根据索引一一对应）。
  - 对应于版本化哈希的 KZG `commitments` ，即 `kzg_to_versioned_hash(commitments[i]) == tx_payload_body.blob_versioned_hashes[i]` 。
  - KZG `commitments` 匹配对应的 `blobs` 和 `proofs`（这可以使用 `verify_blob_kzg_proof_batch` 进行优化，并在从**承诺**和每个 **blob** 的 blob 数据派生的点处进行随机评估的证明）

- 第 2 种：`BlockBodies`（**区块体检索**），使用 **[EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)** 标准 **blob 交易**的 `TransactionPayloadBody` 。

❗ 节点**不可自动**向其对等节点广播 **blob 交易**。相反，这些交易仅使用 `NewPooledTransactionHashes` 消息来公布，然后可以通过 `GetPooledTransactions` 手动请求。这种设计减少了不必要的网络负担，确保只有真正需要这些数据的节点才会请求和接收 **blob 交易** 的详细内容。

---

### 11. **用于验证 blob 交易 KZG 承诺的方法**

整个 EIP 使用相应[共识 4844 规范](https://github.com/ethereum/consensus-specs/blob/86fb82b221474cc89387fa6436806507b3849d88/specs/deneb)中定义的加密方法和类。

具体来说，使用 `[polynomial-commitments.md](https://github.com/ethereum/consensus-specs/blob/86fb82b221474cc89387fa6436806507b3849d88/specs/deneb/polynomial-commitments.md)` 中的以下方法：

- [verify_kzg_proof()](https://github.com/ethereum/consensus-specs/blob/86fb82b221474cc89387fa6436806507b3849d88/specs/deneb/polynomial-commitments.md#verify_kzg_proof)：验证一个给定的  **KZG 承诺** 的单个证明。
- [verify_blob_kzg_proof_batch()](https://github.com/ethereum/consensus-specs/blob/86fb82b221474cc89387fa6436806507b3849d88/specs/deneb/polynomial-commitments.md#verify_blob_kzg_proof_batch)：允许批量验证多个 **KZG 承诺** 的证明。

----

### 12. 辅助方法

1. {**kzg_to_versioned_hash**}：这个函数将一个 KZG 承诺转换为一个版本化哈希值：

   ① 接收一个 `KZGCommitment` 类型的参数（即  KZG 承诺）

   ② 使用 `SHA-256` 哈希函数对承诺进行哈希计算

   ③ 将<u>版本哈希前缀 `VERSIONED_HASH_VERSION_KZG`</u> （新增常量，值为`Bytes1(0x01)`）与 <u>哈希结果去掉第一个字节的剩余部分</u>拼接起来，形成一个新的版本化哈希值。

   ```python
   def kzg_to_versioned_hash(commitment: KZGCommitment) -> VersionedHash:
       return VERSIONED_HASH_VERSION_KZG + sha256(commitment)[1:]
   ```

   此方法通过添加一个**版本前缀**和**使用哈希函数**，确保了 KZG 承诺的唯一性和可验证性，同时在数据结构中明确表示了所使用的加密方案版本。

`KZGCommitment`：基础类型为`Bytes48`，用于以加密安全的方式承诺到一组数据。这里的`Bytes48` 指定了承诺的存储格式。执行“KeyValidate”检查确保了这个承诺是有效的 **BLS 密钥**（即这个 KZG 承诺有效且安全）。允许身份点意味着在特定的加密操作中，**零值**（或"没有数据"）也被视为有效的承诺。

2. {**fake_exponential**}：使用**泰勒展开式**近似以 e 为底的指数计算，从而避免使用浮点数运算。

   计算 `factor * e ** (numerator / denominator)` ：

   ```python
   def fake_exponential(factor: int, numerator: int, denominator: int) -> int:
       i = 1
       output = 0
       numerator_accum = factor * denominator
       while numerator_accum > 0:
           output += numerator_accum
           numerator_accum = (numerator_accum * numerator) // (denominator * i)
           i += 1
       return output // denominator
   ```



## D. 解析与展望

### 1. 以太坊的分片之路

目前 **rollup** 使用 `calldata`做数据存储。未来，**rollups** 将只能使用**分片数据**（即“**blob**”），因为**分片数据**会便宜得多。

以太坊基金会承认**rollup** 过程中不可避免地要对其处理数据的方式进行一次重大升级，但确保 **rollup** 只需升级一次。相比降低现有的 `calldata` 的 gas 成本，采用**分片数据**（ 即 **blob** ）的格式是目前以太坊扩容的权宜之计。

**该 EIP 已经完成的工作包括：**

- 一种新的交易类型，其格式与“**完全分片**”中需要存在的格式完全相同。
- **完全分片**所需的所有*执行层* 逻辑。
- **完全分片**所需的所有*执行* / *共识* 交叉验证逻辑
- `BeaconBlock` 验证和数据可用性采样 **blob** 之间的层分离
- **完全分片**所需的大部分 `BeaconBlock` 逻辑
- **对 blobs** 的自我调整的独立**基础费**。

**实现完全分片未来将完成的工作包括：**

- *共识层* **承诺**的低度扩展以允许 2D 采样
- 数据可用性抽样的实际实施
- PBS（提议者/构建者分离），以避免要求单个验证者在一个**时隙**（slot）中处理 32 MB 的数据
- 每个验证者的托管证明或类似的协议内要求，以验证每个块中分片数据的特定部分

该 EIP 还为长期**协议清理**（protocol cleanups）奠定了基础。例如，其（更清洁的）gas 基础费更新规则可以应用于主要基础费计算。

---

### 2. Rollup 如何发挥作用

Rollup 不会将 **Rollup 区块**的数据放入 `calldata` 中，而是期望 Rollup 块的提交者将这些数据放入 `blob` 中。这保证了可用性且比 `calldata` 便宜得多。 Rollup 需要数据至少**可用一次**，且时间足够长，以确保诚实的参与者可以构建 rollup 状态，但不是永远可用。

- **Optimistic Rollups** 只需要在提交*欺诈证明* 时实际提供基础数据。*欺诈证明* 可以以较小的步骤验证*转换*，通过 `calldata` 一次最多加载 `blob` 的几个值。对于每个值，它将提供 ***KZG proof***，并使用**点评估预编译**(point evaluation precompile)来根据之前提交的**版本化哈希**来验证该值，然后像今天一样对该数据执行*欺诈证明* 验证。

- **ZK Rollups** 将为它们的交易或状态增量数据提供两个承诺：

  - **blob 承诺**（协议确保指向可用数据）。
  - **ZK rollup 自己的承诺**（使用 rollup 内部使用的任何证明系统）。

  **blob 承诺**和 **ZK rollup 自己的承诺**使用“**等价证明协议**”，使用**点评估预编译**(point evaluation precompile)来证明两个承诺引用相同的数据。

---

### 3. 版本化哈希 & 预编译返回数据

使用**版本化哈希**（而不是承诺）作为*执行层* 中 **blob** 的引用，以确保与未来更改的**前向兼容性**。通过引入一个新的版本标识符，支持新的数据结构或验证方法，允许**点评估预编译**使用新格式。 Rollup 无需对其工作方式进行任何 EVM 级别的更改；**sequencer** 只需在适当的时间切换到使用新的交易类型即可。

然而，**点评估（point evaluation）**发生在有限域内，并且只有在已知字段**模数**（mudulus）的情况下才能很好地定义。智能合约可以包含一个将承诺版本映射到**模数**的表，但这不允许智能合约考虑对未知**模数**的未来升级。通过允许访问 EVM 内部的**模数** ，可以构建智能合约，以便它可以使用未来的承诺和证明，而无需升级。

通过让**点评估预编译**(point evaluation precompile)直接返回模数和多项式次数，智能合约可以获得执行验证所需的关键信息，无需依赖额外的预编译来获取这些信息，然后调用者就可以使用它。它也是“免费的”，因为调用者可以忽略返回值的这一部分，而不会产生额外的成本——在可预见的未来保持可升级的系统现在可能会使用这条路线。

---

### 4. 吞吐量

 `TARGET_BLOB_GAS_PER_BLOCK` 和 `MAX_BLOB_GAS_PER_BLOCK` 分别对应于每个块 3 个 **blob** (0.375 MB) 的目标和最多 6 个 **blob** (0.75 MB)的数据容量。这些小的初始限制旨在最大限度地减少该 **EIP** 对网络造成的压力，并且随着网络在较大区块下展现出可靠性，预计会在未来的升级中增加。

---

### 5. 内存池问题

**blob 交易**在**内存池**层具有较大的数据量，这会带来内存池 **DoS** 风险，尽管这并不是前所未有的风险，因为这种风险也确实来源于具有大量 `calldata` 的交易。

通过仅广播 **blob 交易**的公告，接收节点将可以控制接收哪些交易以及接收多少交易，从而允许它们将吞吐量限制在可接受的水平。 **EIP-5793** 将通过扩展 `NewPooledTransactionHashes` 公告消息以包含**交易类型**和**交易大小**，为节点提供进一步的细粒度控制。

此外，以太坊官方建议在内存池交易替换规则中加入 1.1 倍 **blob** **基础费**提升要求。

---

### 6. 安全考虑

此 EIP 使每个信标区块的**带宽**（bandwidth）要求最多增加约 0.75 MB（6 个 **blob**）。

若此 EIP 实施后，理论上每个区块的最大容量比目前的最大容量（30M Gas / 每个 `calldata` 字节 16 Gas = 1.875MB）大 40%（0.75 MB / 1.875MB），理论上不会大幅增加最坏情况下的带宽。合并后，区块时间是静态的，而不是不可预测的泊松分布，为大区块的传播提供了保证的时间段。

即使 `calldata` 容量有限，该 EIP 的*持续负载*也比可降低 `calldata` 成本的方案要低得多，因为不需要将 **blob** 存储与`calldata`一样长的时间，持续时间为 `MIN_EPOCHS_FOR_BLOB_SIDECARS_REQUESTS` 个时段（epoch），约为 18 天。


## E. 小结

EIP-4844 标志着以太坊在解决可扩展性障碍和提高整体网络性能的道路上迈出了重要的一步，同时使未来完全分片所需的更新更少。Proto-Danksharding 增加了 blob 数据空间，这将允许更多的数据处理，显著地降低以太坊 L2 的交易成本，提高 Layer2 的交易吞吐量。会进一步带动 Layer2 生态的繁荣。Dencun 升级还会带动去中心化存储、DA 以及 RaaS 等 Infra 赛道的需求。


> 参考资料
> 1.  [EIP-4844: Shard Blob Transactions](https://eips.ethereum.org/EIPS/eip-4844)
> 2. [MT Capital 研报：全面解读以太坊坎昆升级，潜在机会与利好赛道](https://www.techflowpost.com/article/detail_15655.html)
> 3. [ethereum/consensus-specs](https://github.com/ethereum/consensus-specs/tree/86fb82b221474cc89387fa6436806507b3849d88/specs/deneb)
> 4. [以太坊坎昆升级详解](https://www.datawallet.com/zh/%E9%9A%90%E8%94%BD%E6%80%A7/ethereum-cancun-upgrade-explained)
> 5. [EIP-4844 解释](https://www.datawallet.com/zh/%E9%9A%90%E8%94%BD%E6%80%A7/eip-4844-explained)