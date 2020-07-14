> 📚 VERSION: draft-02
>
> ⛳️ STATE  : WIP

# 数据结构定义 Data Struture Defination

TLV
===

`Tag-Length-Value`是YoMo Codec的基本结构，YoMo的内部数据流是由连续的`TVL`结构构成，每一个`TLV`被成为一个`Packet`

## Tag

固定8 bits长度(1 byte):

```
8   7   6                0
+------------------------+
| F | A |     SeqID      |
+------------------------+
```

1. 其最高位`F`(1000_0000)为`类型`标志位，YoMo Codec有两种`Packet`：`NodePacket`和`PrimitivePacket`。对于`NodePacket`，该bit始终为1；对于`PrimitivePacket`，该bit始终为0
1. 次高位`A`（0100_0000）为数组标识位，当该位为1时，表示该节点的Value为Slice类型（类似于JSON数据结构中的数组）
1. 剩余低7 bits为`顺序标识位Sequence Bits`，用于表示该节点的SeqID（类似于JSON数据结构中的Key的作用）。（所有对于一个`NodePacket`，其Sub-Node最多只能有`2^6=64`个）

### `NodePacket`

表示其`Value`中包含至少一个`NodePacket`

### `PrimitivePacket`

表示其`Value`是基础数据类型

## Length

Length描述了该`Packet`的`Value`的字节长度，是[Varint变长整数类型]

## Value

Value存储了该`Packet`的Value，在`decode`时，用户应指明具体数据类型对其解码

[Varint](SPEC-02.md#varint)
