# Y3 Codec 数据格式规范 v202007

## 概览

版本号：Draft-02 (v202101)。该协议定义了Y3 Codec的帧格式。

1. 添加Slice类型
1. 移除Tag中的次高位作为数据标志位
1. Tag是变长类型
1. 去除基础数据类型（Primitive Packet）和节点数据类型（Node Packet）的标记，全部在解码时指定

## 目录

* [设计目标](#design-goals)
* [符号约定](#notational-conventions)
* [TLV格式](#tlv-format)
  * [标签（Tag）](#tag)
  * [长度（Length）](#length)
  * [值（Value）](#value)
  * [TLV格式编码示例](#tlv-example)
* [基础数据类型](#primitive-type-system)
  * [字符串类型（String）](#string)
  * [二进制类型（Binary）](#binary)
  * [变长整数类型（pvarint）](#pvarint)
  * [变长浮点数类型（TODO）](#float)
  * [Slice类型（Slice）](#slice)

## Design Goals

设计目标

* 是一个`faster than real-time`的编码器。
* 二进制格式适合做编解码，尤其是在流式处理过程中的随机访问场景。
* 为在QUIC Transport的使用优化。
* 本文不涉及RPC场景，与连接无关，仅是数据结构定义。

## Notational Conventions

本文档中的数据包和框架图使用了定制的格式，其目的是这种格式是为了描述表述方法，而不是定义具体的协议元素。该文档定义了完整的语义和结构的细节。

先为复合结构（complex fields）命名，然后是一个包含在`{ }`内的字段列表，该列表中的每个字段都由逗号分隔。

各个字段包括长度信息，以及关于固定的字段的指示：值、可选性或重复性。

单个字段使用以下内容符号惯例，所有长度以位（bit）为单位。

`x (A)`:
表示`x`的长度为`A`位

`x (A..B)`:
表示`x`可以是`A`到`B`之间的任意长度。`A`可以省略，表示最小值是`0`，`B`可以省略，表示最大值没有设置上限

`x (?) = C`:
表示`x`是固定长度为`C`位

`x (?) = C..D`:
表示`x`的值在`C`到`D`之间

`[x (E)]`:
`[]`表示该部分是可选的

`x (E) ...`:
表示`x`会重复零次或多次（并且每次城府都是长度为`E`位）

本文档使用大端序（Big-Endian），字段从每个字节的高位开始放置。

按照惯例，单个字段通过使用以下名称来引用一个复杂字段复杂的结构。

例如：

~~~
Example Structure {
  One-bit Field (1),
  7-bit Field with Fixed Value (7) = 61,
  Arbitrary-Length Field (..),
  Variable-Length Field (8..24),
  Field With Minimum Length (16..),
  Field With Maximum Length (..128),
  [Optional Field (64)],
  Repeated Field (8) ...,
}
~~~

## TLV Format

```txt
0        7
+--------+
| Tag    |
+--------+--------+--------+--------+
| Length                            |
+--------+--------+--------+--------+
| ...
+--------+--------+--------+--------+
| Value Payloads                    |
+--------+--------+--------+--------+
| ...
+--------+--------+--------+--------+
```

~~~
Base Packet {
  Tag (8..),
  Length (8..),
  [Value (8..)],
}

Tag {
  ContinuationBit (1),
  SequenceID (7),
}
~~~

### Tag

标签（Tag）是变长类型，最高位用于标识后续字节是否为连续

```
8   7   6                0
+------------------------+
| F |       SeqID        |
+------------------------+
```

1. 最高位`C`是`连续标识位（Continuation Bit）`，该位为`1`时表示下一个字节（byte）也是该值的一部分，需要继续读取；该位为`0`时表示该字节是整个数值的最后一个字节。
2. 剩余低7位为`顺序ID标识位（Sequence Bits）`，用于表示该节点的`顺序ID（SeqID）`（类似于JSON数据结构中的Key的作用）。

TODO：Tag要支持使用[PVarUInt64](#PVarUInt64)或Raw Bytes的方式编码和解析：

USE CASE：

可以使用`0xFF 0x7F`(18446744073709551615) 作为Tag的值，如果使用[PVarUInt64](#PVarUInt64)对其解码，得到的结果是`0x7F`，而不是`0xFF 0x7F`。即`0xFF 0x7F`与`0x7F`都能被解码成同样的值，但又保证了不同。

#### Supported Types

表示其`值（Value）`的数据类型是`基础数据类型`。

#### Node Type

表示其`值（Value）`包含至少一个`节点类型数据包（NodePacket）`或`原始类型数据包（PrimitivePacket）`，所有子节点按照`Tag-Length-Value`编码依次组合成该`节点类型数据包（NodePacket）`的最终值（Value）。

### Length

`值位长（Length）`描述了该`数据包（Packet）`的`值（Value）`的字节长度，是[变长整数类型](#PVarUInt64)。

### Value

`值（Value）`存储了该`数据包（Packet）`的值内容，在`解码（decode）`时，再指明具体数据类型。

### TLV Example

示例：如果使用`Y3`编码表示下面的`JSON`数据结构

```json
{
    "age" : 5,
    "summary": {
      "name": "CELLA",
      "create": "Y3"
    }
}
```

首先，定义整个消息的结构，就像ProtoBuffer的`.proto`文件做的一样：

```
Tag定义为0x01，表示”age"，其值为变长整型（pvarint）
Tag定义为0x02，表示"summary"，包含2个TLV
  Tag定义为0x03，表示"name"，值为字符串（string）类型
  Tag定义为0x04，表示"create"，值为字符串（string）类型
```

1. 使用`Tag = 0x01`描述`key=age`，其值是`2`，使用变长整型 [pvarint] 类型编码。
2. 使用`Tag = 0x02`描述`key=summary`，其长度（Length）和值（Value）需要从后续步骤推断。
3. 使用`Tag = 0x03`描述`key=name`，其值是"CELLA"，使用字符串 [string] 类型对其编码。
4. 使用`Tag = 0x04`描述`key=create`，其值是"Y3"，使用字符串 [string] 类型对其编码。

编码顺序（序号表示顺序）:

```
1️⃣ 0x01 -> Tag=0x01 描述了 key="age"
    3️⃣ 0x01 -> 该值的长度为1个字节，所以后续1个字节就是该值的具体内容【对于变长整型（pvarint），0x01表示1】
      2️⃣ 0x05 -> 是变长整型（pvarint），0x05是数字5的编码
8️⃣ 0x82 -> Tag=0x82 描述了 key="summary"
    7️⃣ 0x0B -> 该值的长度为11个字节，所以后续11个字节就是该值的具体内容【对于变长整型（pvarint），0x0B表示11】
      1️⃣ 0x03 -> Tag=0x03 描述了 key="name"
        3️⃣ 0x05 -> 该值的长度为5个字节，所以后续5个字节就是该值的具体内容【对于变长整型（pvarint），0x05表示5】
          2️⃣ 0x43 0x45 0x4C 0x4C 0x41 -> "CELLA"的UTF-8编码
      4️⃣ 0x04 -> Tag=0x04 描述了 key="create"
        6️⃣ 0x02 -> 该值的长度为2个字节，所以后续2个字节就是该值的具体内容【对于变长整型（pvarint），0x02表示2】
          5️⃣ 0x59 0x33 -> UTF-8 "Y3"的UTF-8编码
```

最终表示为：

`0x01 0x01 0x05 0x02 0x0B 0x03 0x05 0x43 0x45 0x4C 0x4C 0x41 0x04 0x02 0x59 0x33`

## Primitive Type System

基础数据类型：

### String

使用`UTF-8`编码

### Binary

二进制原始数据

### PVarint

变长整型，`p-var-int` 描述了一种长度可变的整型数值编码：

- `p` 表示`符号位填充（padding signed bit）`。
- `var` 表示`长度可变（variable-length）`。
- `int` 表示`整型（integer）`。

```
8   7   6                0
+---+---+----------------+
| C |(S)|    payloads    |
+---+---+----------------+
```
 
- 使用大端序（Big-Endian）编码
- 最高位`C`是`连续标识位（Continuation Bit）`，该位为`1`时表示下一个字节（byte）也是该值的一部分，需要继续读取；该位为`0`时表示该字节是整个数值的最后一个字节。
- 对于`有符号整数（Signed-Integer）`，次高位`S`是`符号位（Signed Bit）`；对于`无符号整数（Unsigned-Integer）`，该位是数据位。
- 与符号位相同的连续最高位只保留一位，剩余位（bits）使用符号位填充

~~~
pvarint for signed-integer {
  Continuation Bit (1),
  Signed Bit (1),
  Payloads (6..),
}

pvarint for unsigned-integer Value {
  Continuation Bit (1),
  Payloads (7..),
}
~~~

#### Sub types

##### PVarUInt64

##### PVarInt64

##### PVarUInt32

##### PVarInt32

#### Pvarint Example

以`Rust`中的`i32`类型的十进制数`511`为例，其二进制表示为：`0000 0000 0000 0000 0000 0001 1111 1111`，使用了`4个字节`。如果使用[pvarint]类型编码，将分为以下4个步骤：
 
1. 是有符号整数，其有效数据位是：`xxxx xxx0 1111 1111`（为了表示每个字节是8位，使用`x`表示忽略的数据位；`511`是正数，其符号位是`0`）
2. 将所有的`x`位使用符号位`0`填充：`0000 0001 1111 1111`
3. 因为最高位用以表示连续位，所以有效数据位只有7位，我们将每个字节的最高位都插入连续位`y`：`y000 0011 y111 1111`
4. 如果后续还有字节，则该字节的连续标识位（Continuation Bit）为`1`，否则为`0`，因此得到：`1000 0011 0111 1111`

使用`Y3`编码后，只需要2个字节就表示了`511`。

以`Rust`中的`i32`类型的十进制数`-1`为例，其二进制表示为：`1111 1111 1111 1111 1111 1111 1111 1111`，使用了`4个字节`。如果使用[pvarint]类型编码，将分为以下4个步骤：
 
1. 是有符号整数，其有效数据位是：`xxxx xx11`（为了表示每个字节是8位，使用`x`表示忽略的数据位；`-1`是负数，其符号位是`1`）
2. 将所有的`x`位使用符号位`1`填充：`1111 1111`
3. 因为最高位用以表示连续位，所以有效数据位只有7位，我们将每个字节的最高位都插入连续位`y`：`y111 1111`
4. 后续没有字节，则该字节的连续标识位（Continuation Bit）为`0`，因此得到：`0111 1111`

使用`Y3`编码后，只需要1个字节就可表示`-1`。

#### Boolean

布尔类型可以使用 [pvaruint] 类型描述，`1`表示`True`，`0`表示`False`。

### Float

TODO

### Slice

其结构为`Length-Value`的重复。

~~~
Slice {
  Repeated SliceElement (8..) 
}

SliceElement {
  Length with pvarint type (8..),
  Value (8..),
}
~~~
