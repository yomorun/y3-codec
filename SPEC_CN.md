# Y3 Codec: 数据格式规范

## 概览

版本号：Draft-01 (v202007)。该协议定义了Y3 Codec的帧格式。

## 目录

* [设计目标](#design-goals)
* [符号约定](#notational-conventions)
* [TLV格式](#tlv-format)
  * [Tag](#tag)
    * [NodePacket](#nodepacket)
    * [PrimitivePacket](#primitivepacket)
  * [Length](#length)
  * [Value](#value)
  * [示例](#tlv-example)
* [数据类型](#type-system))
  * [字符串类型](#string)
  * [二进制类型](#binary)
  * [变长整数类型](#pvarint)
  * [变长浮点数类型](#TODO)

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
  Tag (8),
  Length (8..),
  Value (8) ...,
}

Tag {
  TypeFlag (1),
  ArrayFlag (1),
  SequenceID (6),
}
~~~

### Tag

标签（Tag）的长度是固定8 bits（1 byte）

```
8   7   6                0
+------------------------+
| F | A |     SeqID      |
+------------------------+
```

1. 其最高位`F`（1000_0000）为`数据包类型（Packet Type）`标志位，Y3 Codec有两种`数据包（Packet）`：`原始类型数据包（PrimitivePacket）`和`节点类型数据包（NodePacket）`。
  1. 对于`原始类型数据包（PrimitivePacket）`，该bit始终为`0`。
  1. 对于`节点数据包（NodePacket）`，该bit始终为`1`。
1. 次高位`A`（0100_0000）为`数组标识位`，当该位为`1`时，表示该节点的`值（Value）`为`切片（Slice）`类型（就像JSON中的数组概念）。
1. 剩余低6位为`顺序ID标识位（Sequence Bits）`，用于表示该节点的`顺序ID（SeqID）`（类似于JSON数据结构中的Key的作用）。

### Length

`值位长（Length）`描述了该`数据包（Packet）`的`值（Value）`的字节长度，是[变长整数类型](#pvarint)

### Value

`值（Value）`存储了该`数据包（Packet）`的值内容，在`解码（decode）`时，再指明具体数据类型。

#### Primitive Types

表示其`值（Value）`的数据类型是`基础数据类型`

#### Node Type

表示其`值（Value）`包含至少一个`节点类型数据包（NodePacket）`或`原始类型数据包（PrimitivePacket）`，所有子节点按照`Tag-Length-Value`编码依次组合成该`节点类型数据包（NodePacket）`的最终值（Value）

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

首先，定义整个消息的结构，就像`.proto`文件做的一样：

```
Primitive Packet, Tag=0x01 -> "age", value type is pvarint
Node Packet, Tag=0x82 -> "summary", this node contains two primitive packets:
  Primitive Packet, Tag=0x03 -> "name", value type is string
  Primitive Packet, Tag=0x04 -> "create", value type is string

```

1. Use `Tag = 0x01` to describe `age`, its value is an integer `2`, use `pvarint` type when encoding
2. Use `Tag = 0x82` to describe `summary`, its value payload need to parsing out
3. Use `Tag = 0x03` to describe `name`, its value is string `CELLA`
4. Use `Tag = 0x04` to describe `create`, its value is string `Y3`

Encoding:

```
0x01 -> Tag=0x01 means key="age" primitive packet
    0x01 -> The length of value is 1 (pvarint type, 0x01=1, means the following 1 byte is value payload)
      0x05 -> 0x05 is pvarint type which represents integer 5
0x82 -> Tag=0x82 means key="summary" node packet
    0x0B -> The length of value is 11 (pvarint type, 0x0B=11, means the following 11 bytes are value payload)
      0x03 -> Tag=0x03 means key="name" primitive packet
        0x05 -> The length of value is 5 (pvarint type, 0x05=5, means the following 5 bytes are value payload)
          0x43 0x45 0x4C 0x4C 0x41 -> UTF-8 string for "CELLA"
      0x04 -> Tag=0x04 means key="create" primitive packet
          0x02 -> The length of value is 2 (pvarint type, 0x02=2, means the following 2 bytes are value payload)
            0x59 0x33 -> UTF-8 string for "Y3"
```

Will be encoded as:

`0x01 0x01 0x05 0x82 0x0B 0x03 0x05 0x43 0x45 0x4C 0x4C 0x41 0x04 0x02 0x59 0x33`

## Type System

支持一下数据类型编码：

1. String
1. Binary
1. Boolean
1. PVarInt32
1. PVarUInt32
1. PVarInt64
1. PVarUInt64
1. VarFloat32
1. VarFloat64

### String

UTF-8 string

### Binary

The raw bytes

### Pvarint

`P-var-int` represents a variable-length integer encoding:

* 'P' is for 'padding signed bit'.
* 'var' is for 'variable-length'.
* 'int' is for 'integer'.

```
8   7   6                0
+---+---+----------------+
| C |(S)|    payloads    |
+---+---+----------------+
```
 
+ Big-Endian
+ C `0x80` represents as `Continuation Bit`, if this bit is `1`, means the following byte need to read next, if this bit is `0`, means
this is the last byte of the value.
+ (S) `0x40` represents as `Signed Bit` for Signed-Integer. for Unsigned-Integer, this is the data bit.
+ 与符号位相同的连续最高位只保留一位，剩余bits使用符号位填充

~~~
PVarInt32 Value {
  Continuation Bit (1),
  Signed Bit (1),
  Payloads (6..),
}

PVarUInt32 Value {
  Continuation Bit (1),
  Payloads (7..),
}
~~~

#### Pvarint Example

An `i32` value `511` in Dec we represent in binary is `0000 0000 0000 0000 0000 0001 1111 1111`, it uses 4 bytes. When we encode it in Pvarint, there are 4 steps:
 
1. The valid bytes are `xxxx xxx0 1111 1111`
2. Padding all the `x` as the signed bit `0`: `0000 0001 1111 1111`
3. Choose 7-bits as value bits: `y000 0011 y111 1111`
4. Change MSB as `1` except the last byte: `1000 0011 0111 1111`

An `i32` value `-1` in Dec we represent in binary is `1111 1111 1111 1111 1111 1111 1111 1111`, it uses 4 bytes. When we encode it in Pvarint, there are 4 steps:
 
1. The valid bytes are `xxxx xx11`
2. Padding all the `x` as the signed bit `1`: `1111 1111`
3. Choose 7-bits as value bits: `y111 1111`
4. Change MSB as `1` except the last byte: `0111 1111`

### Boolean

a `PVarUInt32` value represents `0` OR `1` in 1 byte
