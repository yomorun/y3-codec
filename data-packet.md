## 数据结构定义 Data Struture Defination 

有两种数据结构：

1. NodePacket：描述了一个节点
1. PrimitivePacket：描述具体值类型

NodePacket
==========

NodePacket是`Tag-Length-Value`结构

#### Tag

固定8 bits长度(1 byte):

```
8   7   6                0
+------------------------+
| 1 | A |     SeqID      |
+------------------------+
```

1. 其最高位(1000_0000)为节点标志位，该位始终为1
1. 次高位`A`（0100_0000）为集合标识位，当该位为1时，表示该节点的Value为Slice类型（类似于JSON数据结构中的数组）
1. 剩余低7 bits为数据位，用于表示该节点的SeqID（类似于JSON数据结构中的Key的作用）

#### Length

Length描述了该Node的Value的字节长度，是[Varint]类型

#### Value

Value描述了该Node的Value的内容，它可能是两种形式：`节点(NodePacket)`或`数据(PrimitivePacket)`


PrimitivePacket
===============

PrimitivePacket 用于描述值类型的数据，是`Tag-Length-Type-Value`格式（TLTV）

#### Tag

固定为8 bits长度(1 byte):

```
8   7                    0
+------------------------+
| 0 |       SeqID        |
+------------------------+
```

1. 其最高位始终为0
1. 低7位用于描述该Packet的序号（比如：JSON结构中的：`{a: 1, b: 2, c: 3}`的`a,b,c`可以被分别映射成的Tag为`0x01, 0x02, 0x03`（TODO:用户需提供Key<->SeqID的映射表？）

#### Length

Length描述了`Type+Value`的字节长度，是[Varint]类型

#### Type

固定为8 bits长度(1 byte):

```
8                        0
+------------------------+
|       Data Type        |
+------------------------+
```

已定义的Data Type：

```
// String type data
String = 0x00
// Varint 是可变长度的整数类型
Varint = 0x01
// Float is IEEE754 format as big-endian
Float = 0x02
// Boolean is True OR false
Boolean = 0x03
// UUID is 128-bits fixed-length
UUID = 0x04
// Binary 二进制数据
Binary = 0x40
```

#### Value

Value描述了该PrimitiveNode的Value的内容

[Varint](SPEC.md#varint)
