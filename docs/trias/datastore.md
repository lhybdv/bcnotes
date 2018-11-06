# Trias 数据存储

## 1. 两个概念

### 1.1 条目

以往的区块链如比特币，以太坊，块体里面的结构的格式是固定的。不管是 UTXO 的形式，还是智能合约的形式，格式上都是已经约定好的。为了能够在数据保存上处理的更加灵活，我们提出条目的概念。区块体中存储着的是条目的集合。

#### 1.1.1 什么是条目

条目是一个很宽泛的概念。它是一个格式自解释的数据块，是各种交易形式的一种抽象。格式如下:

| 大小(字节) |    名称     | 数据类型 |   描述   |
| ---------- | ----------- | -------- | -------- |
| 4          | itemType    | uint32   | 类型     |
| varint     | payloadSize | uint32   | 载荷大小 |
| varies     | payload     | []byte   | 载荷     |

对每一种类型，payload 都对应着不同的结构。

#### 1.1.2 条目的优点

1. 不管是 UTXO 的形式，还是智能合约的形式，只要定义好了结构和相应的解析逻辑，把它们作为条目的 payload 部分，从外面看都是一个条目，这就为不同的业务逻辑存储成相同形式带来了可能。

2. 如果现有的数据格式不足以满足新增的业务逻辑，那么可以定义新的数据格式，并定义对应的 itemType，如果已有的格式需要更新，也可以定义基于原来格式的新格式，毕竟区块链的数据是不能篡改的，同时，这也符合了面向对象设计中的 "对修改关闭，对扩展开放" 的开闭原则。是系统可以向后兼容。

### 1.2 文件仓库

比特币，超级账本中都是以区块为单位，多个区块保存到一个区块文件中。EOS 在与 IPFS 的结合中，把交易保存成文件，区块中保存了对交易的引用。显然 EOS 的做法，更加灵活。因为各交易被打包到一个区块中，只是时间相近，所以在存储中物理相邻，它们多数情况下并没有逻辑上的必然联系。所以我们提出文件仓库的概念，主要用来存储条目(也有其他文件，如状态文件等)，一个条目一个文件，区块中只保留文件的索引，索引有可能是 IPFS 的 FileId，或其他。

#### 1.2.1 为什么不直接用 IPFS，还要提出文件仓库

文件仓库是一个抽象，IPFS 只是真实存储文件的方式之一，我们可以使用 IPFS，也可以使用其他类似系统，甚至自己实现，或是几种方式的结合。对于想要获取条目内容的节点来说，可以不必关心实现方式。同时，为今后系统的扩容带来了更多的选择。

#### 1.2.2 为什么不存储区块文件，而是存储条目文件

如前面所说，多个条目逻辑上很可能没有任何关系，假设一个节点想要获取某个条目的内容，而首先获取了整个区块，然后再进行解析，势必加大了无效的数据传输，而影响了系统性能。

## 1. 存储结构

### 1.1 文件系统

文件系统用来存储区块数据，状态数据，历史数据，以二进制数据文件或文本文件来存储。

#### 1.1 文件存储的优势

1. 不受平台影响(操作系统)
2. 不需要更多依赖(相对于数据库来说)
3. 便于与 IPFS 这种去中心化文件系统进行对接
4. 适当的对文件大小进行规划，对文件进行组织，便于传播

#### 1.2 基础数据结构

##### 1.2.1 Block

| 大小(字节) |    名称     | 数据类型 |             描述              |
| ---------- | ----------- | -------- | ----------------------------- |
| 4          | magicNumber | uint32   | 值 待定, 作为区块之间的分隔符 |
| 4          | blockSize   | uint32   | 后面数据到块结束的字节数      |
| 68         | blockHeader | []byte   | 块头 (大小待定)               |
| varint     | itemCount   | uint64   | 条目数量                      |
| varies     | items       | []byte   | 条目集合                      |

##### 1.2.2 Header

| 大小(字节) |        名称         | 数据类型 |               描述                |
| ---------- | ------------------- | -------- | --------------------------------- |
| 4          | version             | uint32   | 版本号                            |
| 32         | previous_block_hash | [32]byte | 前一个 block 的 hash 值           |
| 32         | merkle_root_hash    | [32]byte | 区块内所有 Item 的 merkle_hash 值 |

!!! todo "Header 中的其他项待定"
    比如比特币中的 Hash 难度值，计算用的随机数 Nonce 跟共识算法相关，这里待定

##### 1.2.3 条目


!!! note "条目 (Item)"
    条目 是一个宽泛的概念，它可以是类似比特币的交易，也可以是类似以太坊的交易，这样可以带来两个好处：

    1. 使得账号状态，UTXO 两种不同机制的数据可以共存
    2. 对已有结构进行改进，可以重新定义一个 itemType 及其解释方式

### 1.2 索引系统

用来快速定位存储内容。通过索引可以快速定位到 Block，Item (各种交易或其他数据形式)，以及 Item 当中子项。索引系统使用 levelDB。

## 2 节点分工

```plantuml
@startuml

skinparam {
    backgroundColor #eeeeee
}

[IPFS] as IPFS

package "全节点" as FullNodes {
    [全节点2] as FullNode2
    [全节点1] as FullNode1
}

package "轻节点" as LightNodes {
    [轻节点4] as LightNode4
    [轻节点3] as LightNode3
    [轻节点2] as LightNode2
    [轻节点1] as LightNode1
}

package "客户节点" as ClientNodes {
    [客户节点4] as ClientNode4
    [客户节点3] as ClientNode3
    [客户节点2] as ClientNode2
    [客户节点1] as ClientNode1
}

note left of IPFS: 存放所有区块文件，状态文件
IPFS <-- FullNode1
IPFS <-- FullNode2

note top of FullNodes
    1. 存放 IPFS FileId 与区块哈希的对应关系
    2. 存放所有区块索引，条目索引，状态索引
    3. 验证
    4. 上传区块文件到 IPFS
    5. 缓存近期区块
end note
FullNode1 <-- LightNode1
FullNode1 <-- LightNode2

note top of LightNodes
    1. 存放所有条目哈希
    2. 按照客户节点的交易请求形成条目
    3. 部分验证
    4. 打包区块，上传给全节点
end note
FullNode2 <-- LightNode3
FullNode2 <-- LightNode4

note top of ClientNodes
    1. 存放与本节点相关的信息
    2. 发送交易请求给轻节点
end note
LightNode1 <-- ClientNode1
LightNode2 <-- ClientNode2
LightNode3 <-- ClientNode3
LightNode4 <-- ClientNode4


@enduml
```

### 2.1 全节点

1. 与 IPFS 文件系统进行对接，存储 IPFS 返回的 FileId，形成 FileId 与区块 Hash 的映射关心 
2. 存放所有区块索引，条目索引，状态索引
3. 能提供全面的验证
4. 上传区块文件到 IPFS
5. 缓存近期区块

!!! note
    IPFS 只负责存储文件，并不参与业务逻辑，如何解析文件内容，还需要全节点来负责。
    IPFS 的 FileId 虽然也是内容哈希，但与区块哈希的计算规则并不相同，所以需要保存映射关系

### 2.2 轻节点

1. 存放所有条目哈希，在接到客户节点交易请求时可根据条目哈希从全节点获取条目数据，进行验证
2. 按照客户节点的交易请求形成条目
3. 部分验证
4. 打包区块，上传给全节点

### 2.3 客户节点

1. 保存与本节点相关的信息，最大化减少数据存储量，增强客户节点的可用性

## 3 同步

### 3.1 被动同步

#### 3.1.1 被动同步时机

#### 3.1.1.1 NewBlockHashMsg

全节点

```mermaid
sequenceDiagram
participant L as Local FullNode
participant P as Peer FullNode

P ->> L: NewBlockHashMsg
L ->> L: CheckHashExist
L ->> P: GetBlockMsg
P -->> L: BlockMsg
L ->> L: SaveBlock Send NewBlockHashMsg

```

过程描述:

1. Peer FullNode 在存入新区块，或获取新区块之后 发出 NewBlockHashMsg
2. Local FullNode 获取到 NewBlockHashMsg，在本地进行对比，若不存在，则发出 GetBlockMsg 给 Peer FullNode
3. Peer FullNode 获取到 GetBlockMsg 后，从本地查询并通过 BlockMsg 将 Block 信息发送给 Local FullNode
4. Local FullNode 在拿到 Block 信息之后，计算 Block，Item 等信息的索引，并将 Block，相关索引保存

轻节点

```mermaid
sequenceDiagram
participant L as Local LightNode
participant P as Peer FullNode

P ->> L: NewBlockHashMsg
L ->> L: CheckHashExist
L ->> P: GetBlockHashMsg
P -->> L: BlockHashMsg
L ->> L: SaveHash SaveItemHash
```


过程描述:

1. Peer FullNode 在存入区块，或获取新区块之后 发出 NewBlockHashMsg
2. Local LightNode 获取到 NewBlockHashMsg，在本地进行对比，若不存在，则发出 GetBlockHashMsg 给 Peer FullNode
3. Peer FullNode 通过 BlockHashMsg 将 BlockHash, ItemHash 等信息发送给 Local ClientNode
4. Local ClientNode 保存 BlockHash, ItemHash 等信息

!!! note "BlockHashMsg"
    BlockHashMsg 会包含区块当中的 条目 Hash


## 3.2 主动同步

### 3.2.1 主动同步时机

#### 3.2.1.1 新节点加入

全节点

```mermaid
sequenceDiagram
participant L as Local FullNode
participant P as Peer FullNode

L ->> P: GetHeaderHashMsg
P -->> L: HeaderHashMsg
L ->> P: GetBlockMsg
P -->> L: BlockMsg
L ->> L: SaveBlock
```

过程描述:

1. 当一个新的全节点加入，会循环向其他 FullNode 发出 GetHeaderHashMsg
2. 远端 FullNode 发送 HeaderHashMsg 给本节点
3. 本节点根据 HeaderHash 向远端节点 发出 GetBlockMsg 
4. 远端 FullNode 发送 BlockMsg 给本节点
5. 本节点进行数据保存

轻节点

```mermaid
sequenceDiagram
participant L as Local LightNode
participant P as Peer LightNode

L ->> P: GetHeaderHashMsg
P -->> L: HeaderHashMsg
L ->> L: SaveHash
```

过程描述:

1. 当一个新的全节点加入，会循环向其他 FullNode 发出 GetHeaderHashMsg
2. 远端 FullNode 发送 HeaderHashMsg 给本节点
5. 本节点进行数据保存