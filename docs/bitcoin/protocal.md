# 比特币通信协议

## 1 共用结构

### 1.1 Message(消息)

| 字段尺寸 |   描述   | 数据类型 |                                说明                                 |
| -------- | -------- | -------- | ------------------------------------------------------------------- |
| 4        | magic    | uint32_t | 用于识别消息的来源网络，当流状态位置时，它还用于寻找下一条消息      |
| 12       | command  | char[12] | 识别包内容的ASCII字串，用NULL字符补满，(使用非NULL字符填充会被拒绝) |
| 4        | length   | uint32_t | payload的字节数                                                     |
| 4        | checksum | uint32_t | sha256(sha256(payload)) 的前4个字节(不包含在version 或 verack 中)   |
| ?        | payload  | uchar[]  | 实际数据                                                            |

!!! note "command"
    1. char 范围 [-128, 127], 第 1 位表示符号，0 为正，1 为负
    2. uchar[] 范围 [0, 255]
    3. command 就是消息的类型，目前涉及的类型有：
        1. version
        2. verack
        3. addr
        4. inv
        5. getdata
        6. getblocks
        7. getheaders
        8. tx
        9. block
        10. headers
        11. getaddr
        15. ping
        16. alert
        17. merkleblock
        18. mempool
        19. pong
        20. notfound
        21. filterload
        22. filteradd
        23. filterclear
        24. reject
        25. sendheaders
        26. feefilter
        27. sendcmpct
        28. cmpctblock
        29. getblocktxn
        30. blocktxn

!!! note checksum
    1. 对 payload (消息体) 的 2 次 sha256 哈希之后的前 4 个字节，跟 git 用法类似，前几位足以起到区别的作用
    2. 应该 *不是* 用于防恶意篡改的，因为发出方自己可以重新计算，同时改 payload 和 checksum 
    3. 应该 *是* 防止网络传输、读取错误之类的导致数据不完整
    4. version, verack 这 2 个消息没有 checksum，因为没有 payload

已定义的magic值：

| Network  | Magic value | Sent over wire as |
| -------- | ----------- | ----------------- |
| main     | 0xD9B4BEF9  | F9 BE B4 D9       |
| testnet  | 0xDAB5BFFA  | FA BF B5 DA       |
| testnet3 | 0x0709110B  | 0B 11 09 07       |
| namecoin | 0xFEB4BEF9  | F9 BE B4 FE       |

#### 1.2.1.2 Variable length integer (变长整数)

| 值            | 存储长度 | 格式            |
| ------------- | -------- | --------------- |
| < 0xfd        | 1        | uint8_t         |
| <= 0xffff     | 3        | 0xfd + uint16_t |
| <= 0xffffffff | 5        | 0xfe + uint32_t |
| -             | 9        | 0xff + uint64_t |

#### 1.2.1.3 Variable length string (变长字符串)

| 字段尺寸 |  描述  | 数据类型 |        说明        |
| -------- | ------ | -------- | ------------------ |
| ?        | length | var_int  | 字符串长度         |
| ?        | string | char[]   | 字符串本身(可为空) |

#### 1.2.1.4 Network address (网络地址)

| 字段尺寸 |   描述   | 数据类型 |               说明               |
| -------- | -------- | -------- | -------------------------------- |
| 8        | services | uint64_t | 与version 消息中的service(s)相同 |
| 16       | IPv6/4   | char[16] | Ipv6地址，以网络字节顺序存储。   |
| 2        | port     | uint16_t | 端口号，以网络字节顺序存储。     |

!!! note IP 地址
    * 官方客户端仅支持IPv4，仅读取最后4个字节以获取IPv4地址。
    * IPv4地址以16字节的IPv4映射位址格式写入结构。(12字节 00 00 00 00 00 00 00 00 00 00 FF FF, 后跟4 字节IPv4地址)

#### 1.2.1.5 Inventory Vectors (清单向量)

| 字段尺寸 | 描述 | 数据类型 | 说明         |
| -------- | ---- | -------- | ------------ |
| 4        | type | uint32_t | 对象类型标识 |
| 32       | hash | char[32] | 对象散列值   |

目前对象类型标识已经定义如下3个值

| Value |        Name        |                                                                                                           Description                                                                                                            |
| ----- | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0     | ERROR              | Any data of with this number may be ignored                                                                                                                                                                                      |
| 1     | MSG_TX             | Hash is related to a transaction                                                                                                                                                                                                 |
| 2     | MSG_BLOCK          | Hash is related to a data block                                                                                                                                                                                                  |
| 3     | MSG_FILTERED_BLOCK | Hash of a block header; identical to MSG_BLOCK.<br> Only to be used in getdata message.<br> Indicates the reply should be a merkleblock message rather than a block message; <br>this only works if a bloom filter has been set. |
| 4     | MSG_CMPCT_BLOCK    | Hash of a block header; identical to MSG_BLOCK.<br> Only to be used in getdata message.<br> Indicates the reply should be a cmpctblock message. See BIP 152 for more info.                                                               |

#### 1.2.1.6 Block Headers (Block头部)

| 字段尺寸 |    描述     | 数据类型 |                   说明                   |
| -------- | ----------- | -------- | ---------------------------------------- |
| 4        | version     | uint32_t | Block版本信息，基于创建该block的软件版本 |
| 32       | prev_block  | char[32] | 该block前一block的散列                   |
| 32       | merkle_root | char[32] | 与该block相关的全部交易之散列(Merkle树)  |
| 4        | timestamp   | uint32_t | 记录block创建时间的时间戳                |
| 4        | bits        | uint32_t | 创建block的计算难度                      |
| 4        | nonce       | uint32_t | 用于生成block的临时数据                  |
| 1        | txn_count   | uint8_t  | 交易数，这个值总是0                      |

## 2 消息类型

### 2.1 version

一个节点收到连接请求时，它立即宣告其版本。在通信双方都得到对方版本之前，不会有其他通信

|    字段尺寸    |      描述       | 数据类型 |              说明              |
| -------------- | --------------- | -------- | ------------------------------ |
| 4              | version         | uint32_t | 节点使用的协议版本标识         |
| 8              | services        | uint64_t | 该连接允许的特性(bitfield)     |
| 8              | timestamp       | uint64_t | 以秒计算的标准UNIX时间戳       |
| 26             | addr_me         | net_addr | 生成此消息的节点的网络地址     |
| version >= 106 |                 |          |                                |
| 26             | addr_you        | net_addr | 接收此消息的节点的网络地址     |
| 8              | nonce           | uint64_t | 节点的随机id，用于侦测这个连接 |
| ?              | sub_version_num | var_str  | 辅助版本信息                   |
| version >= 209 |                 |          |                                |
| 4              | start_height    | uint32_t | 发送节点接收到的最新block      |

### 2.2 verack

版本不低于209的客户端在应答version消息时发送verack消息

### 2.3 addr

| 字段尺寸 |   描述    |        数据类型         |                      说明                       |
| -------- | --------- | ----------------------- | ----------------------------------------------- |
| 1+       | count     | var_int                 | 地址数                                          |
| 30x?     | addr_list | (uint32_t + net_addr)[] | 网络上其他节点的地址，版本低于209时仅读取第一条 |

### 2.4 inv

| 字段尺寸 |   描述    |  数据类型  |        说明         |
| -------- | --------- | ---------- | ------------------- |
| ?        | count     | var_int    | 清单(inventory)数量 |
| 36x?     | inventory | inv_vect[] | 清单(inventory)数据 |

### 2.5 getdata

| 字段尺寸 |   描述    |  数据类型  |        说明         |
| -------- | --------- | ---------- | ------------------- |
| ?        | count     | var_int    | 清单(inventory)数量 |
| 36x?     | inventory | inv_vect[] | 清单(inventory)数据 |

### 2.6 getblocks

| 字段尺寸 |    描述     | 数据类型 |                           说明                            |
| -------- | ----------- | -------- | --------------------------------------------------------- |
| 1+       | start count | var_int  | hash_start 的数量                                         |
| 32+      | hash_start  | char[32] | 发送节点已知的最新block散列                               |
| 32       | hash_stop   | char[32] | 请求的最后一个block的散列，若要获得尽可能多的block则设为0 |

### 2.7 getheaders

| 字段尺寸 |    描述     | 数据类型 |                           说明                            |
| -------- | ----------- | -------- | --------------------------------------------------------- |
| 1+       | start count | var_int  | hash_start 的数量                                         |
| 32+      | hash_start  | char[32] | 发送节点已知的最新block散列                               |
| 32       | hash_stop   | char[32] | 请求的最后一个block的散列，若要获得尽可能多的block则设为0 |

### 2.8 tx

| 字段尺寸 |     描述     | 数据类型 |                                                                    说明                                                                     |
| -------- | ------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 4        | version      | uint32_t | 交易数据格式版本                                                                                                                            |
| 1+       | tx_in count  | var_int  | 交易的输入数                                                                                                                                |
| 41+      | tx_in        | tx_in[]  | 交易输入或比特币来源列表                                                                                                                    |
| 1+       | tx_out count | var_int  | 交易的输出数                                                                                                                                |
| 8+       | tx_out       | tx_out[] | 交易输出或比特币去向列表                                                                                                                    |
| 4        | lock_time    | uint32_t | 锁定交易的期限或block数目。如果为0则交易一直被锁定。未锁定的交易不可包含在block中，并可以在过期前修改(目前bitcon不允许更改交易，所以没有用) |

tx_in

| 字段尺寸 |       描述       | 数据类型 |                          说明                           |
| -------- | ---------------- | -------- | ------------------------------------------------------- |
| 36       | previous_output  | outpoint | 对前一输出的引用                                        |
| 1+       | script length    | var_int  | signature script 的长度                                 |
| ?        | signature script | uchar[]  | 用于确认交易授权的计算脚本                              |
| 4        | sequence         | uint32_t | 发送者定义的交易版本，用于在交易被写入block之前更改交易 |

outPoint

| 字段尺寸 | 描述  | 数据类型 |                     说明                      |
| -------- | ----- | -------- | --------------------------------------------- |
| 32       | hash  | char[32] | 引用的交易的散列                              |
| 4        | index | uint32_t | 指定输出的索引，第一笔输出的索引是0，以此类推 |

tx_out

| 字段尺寸 |       描述       | 数据类型 |                                              说明                                               |
| -------- | ---------------- | -------- | ----------------------------------------------------------------------------------------------- |
| 8        | value            | uint64_t | 交易的比特币数量(单位是0.00000001)                                                              |
| 1+       | pk_script length | var_int  | pk_script的长度                                                                                 |
| ?        | pk_script        | uchar[]  | Usually contains the public key as a Bitcoin script setting up conditions to claim this output. |

### 2.9 block

| 字段尺寸 |    描述     | 数据类型 |                   说明                    |
| -------- | ----------- | -------- | ----------------------------------------- |
| 4        | version     | uint32_t | block版本信息，基于生成block的软件版本    |
| 32       | prev_block  | char[32] | 这一block引用的前一block之散列            |
| 32       | merkle_root | char[32] | 与这一block相关的全部交易之散列(Merkle树) |
| 4        | timestamp   | uint32_t | 记录block创建时间的时间戳                 |
| 4        | bits        | uint32_t | 这一block的计算难度                       |
| 4        | nonce       | uint32_t | 用于生成这一block的nonce值                |
| ?        | txn_count   | var_int  | 交易数量                                  |
| ?        | txns        | tx[]     | 交易，以tx格式存储                        |

### 2.10 headers

| 字段尺寸 |  描述   |    数据类型    |    说明     |
| -------- | ------- | -------------- | ----------- |
| ?        | count   | var_int        | block头数量 |
| 77x?     | headers | block_header[] | block头     |

### 2.11 getaddr

!!! question getaddr 似乎也没有消息体?

### 2.12 checkorder

|       字段尺寸        |         描述          | 数据类型 | 说明 |
| --------------------- | --------------------- | -------- | ---- |
| Fields from CMerkleTx |                       |          |      |
| ?                     | hashBlock             |          |      |
| ?                     | vMerkleBranch         |          |      |
| ?                     | nIndex                |          |      |
| Fields from CWalletTx |                       |          |      |
| ?                     | vtxPrev               |          |      |
| ?                     | mapValue              |          |      |
| ?                     | vOrderForm            |          |      |
| ?                     | fTimeReceivedIsTxTime |          |      |
| ?                     | nTimeReceived         |          |      |
| ?                     | fFromMe               |          |      |
| ?                     | fSpent                |          |      |

### 2.13 submitorder

| 字段尺寸 |     描述     | 数据类型  |           说明            |
| -------- | ------------ | --------- | ------------------------- |
| 32       | hash         | char[32]  | 交易散列                  |
| ?        | wallet_entry | CWalletTx | 与checkorder的payload相同 |

### 2.14 reply

| 字段尺寸 | 描述  | 数据类型 | 说明     |
| -------- | ----- | -------- | -------- |
| 4        | reply | uint32_t | 应答代码 |

可能值:

| 值 | 名称         | 说明                                                                |
| -- | ------------ | ------------------------------------------------------------------- |
| 0  | SUCCESS      | IP Transaction可以执行(回应checkorder)或已经被接受(回应submitorder) |
| 1  | WALLET_ERROR | AcceptWalletTransaction()失败                                       |
| 2  | DENIED       | 此节点不接受IP Transactions                                         |

### 2.15 ping

### 2.16 alert

| 字段尺寸 |   描述    | 数据类型 |                    说明                     |
| -------- | --------- | -------- | ------------------------------------------- |
| ?        | message   | var_str  | 向网络中所有节点发出的系统消息              |
| ?        | signature | var_str  | 可由公钥验证Satoshi授权或创建了此信息的签名 |