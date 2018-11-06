# EOS 数据结构

|        字段        |                 说明                 |
| ------------------ | ------------------------------------ |
| timestamp          | 时间戳                               |
| producer           | 生产者                               |
| confirmed          | 生产者确认数                         |
| previous           | 链式结构前一个区块的id               |
| transaction_mroot  | 交易默克尔树根                       |
| action_mroot       | 动作默克尔树根                       |
| schedule_version   | 生产者版本排序号                     |
| new_producers      | 下一个生产者                         |
| header_extensions  | 区块头扩展字段                       |
| producer_signature | 区块签名，由生产者签名               |
| transactions       | 块打包交易内容，是数组结构，可以多个 |
| block_extensions   | 区块扩展字段                         |
| id                 | 当前块id                             |
| block_num          | 当前块高度                           |
| ref_block_prefix   | 引用区块的区块头                     |