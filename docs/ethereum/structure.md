# 以太坊数据结构

## 1 以太坊中的序列化方法RLP

>RLP(Recursive Length Prefix)可以将任意的数据编码称二进制byte的数组，即[]byte的形式。同时已知数据的RLP编码结果，可以求出其原来的形式

### 1.1 RLP
[RLR 说明](https://github.com/ethereum/wiki/wiki/RLP)

### 1.2 以太坊中的SHA3计算

```
encode(data)=SHA3(RLP(data))
```

## 2 区块的组成

### 2.1 三部分组成

1. 区块头（Block Header）
2. 叔块（Uncle）
3. 交易列表（tx_List）

!!! note "叔块"
    1. 以太坊每 10 几秒产生一个区块，这样就会有大量的 "孤块" 存在。
    2. 如果 "孤块" 都 *不被* 承认，可能会有很多计算力小的节点退出，导致以太坊趋于 "中心化"，不够安全
    3. "孤块" 如何被承认为 "叔块" 呢? 就是被之后的 "新块" 在打包的时候，包含进去，这样 "新块" 被认可的时候，"叔块" 也得到了认可。
    4. "新块" 为什么会愿意打包 "叔块" 呢，因为有奖励
    5. "叔块" 的创建者也会有奖励，这样，算力小，网速慢 的一些节点也不会 "白忙"，整个系统更 “公平"，更 "去中心化"

### 2.2 区块头结构

|    名称     |      类型      |                                      意义                                      |
| ----------- | -------------- | ------------------------------------------------------------------------------ |
| parentHash  | common.Hash    | 父区块的哈希值                                                                 |
| UncleHash   | common.Hash    | 叔父区块列表的哈希值                                                           |
| Coinbase    | common.Address | 打包该区块的矿工的地址，用于接收矿工费                                         |
| Root        | common.Hash    | 状态树的根哈希值                                                               |
| TxHash      | common.Hash    | 交易树的根哈希值                                                               |
| ReceiptHash | common.Hash    | 收据树的根哈希值                                                               |
| Bloom       | Bloom          | 交易收据日志组成的Bloom过滤器                                                  |
| Difficuty   | *Big.Int       | 本区块的难度                                                                   |
| Number      | *Big.Int       | 本区块块号，区块号从 0 开始算起                                                |
| GasLimit    | uint64         | 本区块中所有交易消耗的 Gas 上限，这个数值*不等于*所有交易中的 GasLimit字段的和 |
| GasUsed     | uint64         | 本区块中所有交易使用的 Gas 的和                                                |
| Time        | *Big.Int       | 区块产生的 unix 时间戳，一般是打包区块的时间，这个字段不是出块的时间戳         |
| Extra       | []byte         | 区块的附加数据                                                                 |
| MixDigest   | common.Hash    | 哈希值，与 Nonce 的组合用于工作量计算                                          |
| Nonce       | BlockNonce     | 区块产生是的随机值                                                             |

### 2.3 MPT

融合了 Merkle Tree，Patricia Tree (源于 Trie Tree)

### 2.3.1 4 种节点

1. fullNode
2. shorNode
3. valueNode
4. hashNode

!!! note "fullNode"
    1. fullNode 是一个可以携带多个子节点的父(枝)节点。
    2. 它有一个容量为17的node数组成员变量Children，数组中前16个空位分别对应16进制(hex)下的0-9a-f，这样对于每个子节点，根据其key值16进制形式下的第一位的值，就可挂载到Children数组的某个位置。
    3. fullNode本身不再需要额外key变量。
    4. Children数组的第17位，留给该fullNode的数据部分。
    5. fullNode明显继承了原生trie的特点，而每个父节点最多拥有16个分支也包含了基于总体效率的考量

!!! note "shorNode"
    1. shortNode 是一个仅有一个子节点的父(枝)节点。
    2. 它的成员变量Val指向一个子节点，而成员Key是一个任意长度的字符串(字节数组[]byte)。
    3. 显然shortNode的设计体现了PatriciaTrie的特点，通过合并只有一个子节点的父节点和其子节点来缩短trie的深度，结果就是有些节点会有长度更长的key。

!!! note "valueNode"
    1. valueNode 充当MPT的叶子节点。
    2. 它其实是字节数组[]byte的一个别名，不带子节点。
    3. 在使用中，valueNode就是所携带数据部分的RLP哈希值，长度32byte，数据的RLP编码值作为valueNode的匹配项存储在数据库里。

!!! note "hashNode"
    1. hashNode 跟valueNode一样，也是字符数组[]byte的一个别名，同样存放32byte的哈希值，也没有子节点。
    2. 不同的是，hashNode是fullNode或者shortNode对象的RLP哈希值，所以它跟valueNode在使用上有着莫大的不同。

