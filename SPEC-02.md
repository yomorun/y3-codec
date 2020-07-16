# YoMo-Codec: Base Format (Invariants)

## Overview

Version Draft-01 (v202007). This protocol defines the framing individual message formats.

## TOC

* [YoMo Codec Packet Format](#yomo-codec-packet-format)
* [TLV Format](#tlv-format)
* [Primitive Types](#primitive-types)
  * [Varint](#varint)

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

[TLTV format spec can see here](data-packet-02.md)

```txt
0        7
+--------+
| Tag    |
+--------+-----—-+-------------+
| Length (varint)
+--------+-------+-------------+
| Value Payload
+------------------------------+
```

~~~
Base Packet {
  MSB (1),
  ArrayFlag (1),
  SequenceID (6),
  Length Varint Type (8..),
  Value (8..),
}
~~~

e.g.: If we want to transform this `JSON` format object as YoMo-Codec:

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
Primitive Packet, Tag=0x01 -> "age", value type is varint
Node Packet, Tag=0x82 -> "summary", this node contains two primitive packets:
  Primitive Packet, Tag=0x03 -> "name", value type is string
  Primitive Packet, Tag=0x04 -> "create", value type is string

```

1. Use `Tag = 0x01` to describe `age`, its value is an integer `2`, use `varint` type when encoding
2. Use `Tag = 0x82` to describe `summary`, its value payload need to parsing out
3. Use `Tag = 0x03` to describe `name`, its value is string `CELLA`
4. Use `Tag = 0x04` to describe `create`, its value is string `Y3`

Encoding:

```
0x01 -> Tag=0x01 means key="age" primitive packet
    0x02 -> The length of value is 1 (Varint type, 0x02=1, means the following 1 byte is value payload)
      0x02 -> 0x02 is varint type which represents integer 1
0x82 -> Tag=0x82 means key="summary" node packet
    0x16 -> The length of value is 11 (Varint type, 0x16=11, means the following 11 bytes are value payload)
      0x03 -> Tag=0x03 means key="name" primitive packet
        0x0A -> The length of value is 5 (Varint type, 0x0A=5, means the following 5 bytes are value payload)
          0x43 0x45 0x4C 0x4C 0x41 -> UTF-8 string for "CELLA"
      0x04 -> Tag=0x04 means key="create" primitive packet
          0x04 -> The length of value is 2 (Varint type, 0x04=2, means the following 2 bytes are value payload)
            0x59 0x33 -> UTF-8 string for "Y3"
```

Will be encoded as:

`0x01 0x0A 0x43 0x45 0x4C 0x4C 0x41 0x02 0x02 0x02`


## Primitive Types

There are 5 meta types in `Type`:

1. `String`: `UTF-8` encoding
2. `Integer`: [Varint](#Varint)
3. `Float`: [IEEE754](https://en.wikipedia.org/wiki/IEEE_754) format as big-endian
5. `Boolen`: fixed-length 1 byte
4. `UUID`: 128-bits fixed-length
6. `Binary`: raw binary data

### Varint

TODO: Use complement from @figroc

> Varint represents a variable-length integer in [LEB128 Encoding](https://google.com/search?q=LEB128+Encoding) format.
> 
> The rules are:
> 
> 1. Uses [zigzag](https://developers.google.com/protocol-buffers/docs/encoding#signed-integers) encoding to transform all signed integers to unsigned integers (`i64` to `u64`), so we don't need to care about the signed integers when encoding/decoding.
> 2. The highest bit of each byte reperesents as MSB (Most-Significant-Bit): when `MSB` is `1`, it indicates the following byte is also a part of `varint`, `0` indicates this byte is last byte of whole varint. The lower 7-bits are used to represent the value.
> 3. Uses little-endian encoding.
> 
> #### Varint Example
> 
> An i32 value `259` in Dec we represent in binary is `0000 0000 0000 0000 0000 0001 0000 0011`, it uses 4 bytes. When we encode it in YoMo-Codec, there are 4 steps:
> 
> 1. The valid bytes are `0000 0001 0000 0011`
> 2. Choose 7-bits as value bits: `0000 0010 0000 0011`
> 3. Use little-endian sequence, transform to: `0000 0011 0000 0010`
> 4. Change MSB as `1` except the last byte: `1000 0011 0000 0010`
