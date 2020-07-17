# YoMo-Codec: Base Format (Invariants)

## Overview

Version Draft-01 (v202007). This protocol defines the framing individual message formats.

## TOC

* [YoMo Codec Packet Format](#yomo-codec-packet-format)
* [Notational Conventions](#notational-conventions)
* [TLV Format](#tlv-format)
  * [Tag](#tag)
    * [NodePacket](#nodepacket)
    * [PrimitivePacket](#primitivepacket)
  * [Length](#length)
  * [Value](#value)
  * [TLV Example](#tlv-example)
* [Pvarint](#pvarint)
  * [Pvarint Example](#pvarint-example)

## YoMo Codec Packet Format

YoMo-Codec defines YoMo message formats.

* Binary is good for encoding/decoding, especially for server side. You can directly skip the data which you don't care at all.
* Not design for saving bytes, aimed for 5G & Wifi6
* Not related to connection protocol, just a message format

## Notational Conventions

Packet and frame diagrams in this document use a bespoke format. The purpose of
this format is to summarize, not define, protocol elements. Prose defines the
complete semantics and details of structures.

Complex fields are named and then followed by a list of fields surrounded by a
pair of matching braces. Each field in this list is separated by commas.

Individual fields include length information, plus indications about fixed
value, optionality, or repetitions. Individual fields use the following
notational conventions, with all lengths in bits:

`x (A)`:
: Indicates that x is A bits long

`x (i)`:
: Indicates that x uses the variable-length encoding in {{integer-encoding}}

`x (A..B)`:
: Indicates that x can be any length from A to B; A can be omitted to indicate
  a minimum of zero bits and B can be omitted to indicate no set upper limit;
  values in this format always end on an octet boundary

`x (?) = C`:
: Indicates that x has a fixed value of C

`x (?) = C..D`:
: Indicates that x has a value in the range from C to D, inclusive

\[x (E)\]:
: Indicates that x is optional (and has length of E)

`x (E) ...`:
: Indicates that x is repeated zero or more times (and that each instance is
  length E)

This document uses network byte order (that is, big endian) values.  Fields
are placed starting from the high-order bits of each byte.

By convention, individual fields reference a complex field by using the name of
the complex field.

For example:

~~~
Example Structure {
  One-bit Field (1),
  7-bit Field with Fixed Value (7) = 61,
  Field with Variable-Length Integer (i),
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
|  Tag   |
+--------+-----—-+-------------+
| Length (pvarint)
+--------+-------+-------------+
| ...
+--------+-------+-------------+
| Value Payloads
+------------------------------+
| ...
+--------+-------+-------------+
```

~~~
Tag {
  MSB (1),
  ArrayFlag (1),
  SequenceID (6),
}

Base Packet {
  Tag (8),
  Length Varint Type (8..),
  Value (8..),
}
~~~

### Tag

固定8 bits长度(1 byte):

```
8   7   6                0
+------------------------+
| F | A |     SeqID      |
+------------------------+
```

1. 其最高位`F`(1000_0000)为`Packet Type`标志位，YoMo Codec有两种`Packet`：`NodePacket`和`PrimitivePacket`。对于`NodePacket`，该bit始终为1；对于`PrimitivePacket`，该bit始终为0
1. 次高位`A`（0100_0000）为数组标识位，当该位为1时，表示该节点的Value为Slice类型（类似于JSON数据结构中的数组）
1. 剩余低7位为`顺序标识位Sequence Bits`，用于表示该节点的SeqID（类似于JSON数据结构中的Key的作用）。（所有对于一个`NodePacket`，其Sub-Node最多只能有`2^6=64`个）

#### NodePacket

表示其`Value`包含至少一个`NodePacket`或`PrimitivePacket`

#### PrimitivePacket

表示其`Value`是基础数据类型

### Length

Length描述了该`Packet`的`Value`的字节长度，是[Pvarint变长整数类型](#pvarint)

### Value

Value存储了该`Packet`的Value，在`decode`时，用户应指明具体数据类型对其解码

### TLV Example

If we want to transform this `JSON` format object as YoMo-Codec:

```json
{
    "age" : 5,
    "summary": {
      "name": "CELLA",
      "create": "Y3"
    }
}
```

First define the message struct, just like a `.proto` file does:

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

## Pvarint

Pvarint represents a variable-length integer encoding. 'P' is for 'padding signed bit'

```
8   7   6                0
+------------------------+
| C |(S)|    payload     |
+------------------------+
```
 
+ Big-Endian
+ C `0x80` represents as `Continuation Bit`, if this bit is `1`, means the following byte need to read next, if this bit is `0`, means
this is the last byte of the value.
+ (S) `0x40` represents as `Signed Bit` for Signed-Integer. for Unsigned-Integer, this is the data bit.
+ 与符号位相同的连续最高位只保留一位，剩余bits使用符号位填充

### Pvarint Example

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
