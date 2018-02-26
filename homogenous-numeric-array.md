# Homogenous Numeric Arrays for CBOR

This document specifies tags for representing Homogenous Numeric Arrays (HNAs) in Concise Binary Object Representation (CBOR) [1].
These tags allow efficient manipulation of large arrays of numeric values.

## Registration information

    Tag: 1100
    Semantics: Homogenous uint16_t array

    Tag: 1101
    Semantics: Homogenous uint32_t array

    Tag: 1102
    Semantics: Homogenous uint64_t array

    Tag: 1104
    Semantics: Homogenous int8_t array

    Tag: 1105
    Semantics: Homogenous int16_t array

    Tag: 1106
    Semantics: Homogenous int32_t array

    Tag: 1107
    Semantics: Homogenous int64_t array

    Tag: 1109
    Semantics: Homogenous half-precision float array

    Tag: 1110
    Semantics: Homogenous single-precision float array

    Tag: 1111
    Semantics: Homogenous double-precision float array

    Data item: byte string
    Point of contact: Jeffrey Tippet <jeffrey.tippet@outlook.com>
    Description of semantics: https://github.com/jtippet/cbor-specs/blob/master/homogenous-numeric-array.md


## Semantics

When tagged with one of the tags listed above, a CBOR byte string MAY be interpreted as a homogenous array of numbers (HNA).
Semantically, an HNA is an array of numbers, each of which has the same numeric data type.
For example, a `uint32_t` HNA is semantically an array of `uint32_t` values, even though it is represented in CBOR as a byte string.

The `i`-th number in an HNA is stored begining at offset `n × i` in the byte string, where `n` is the natural width of the numeric type.
For example, the 3rd value in an `uint32_t` HNA is stored in bytes 16 through 24.

### Representation of numbers

Numeric values MUST be stored in network byte order.

Unlike negative numbers in CBOR Major Type 1, negative values in an HNA MUST be stored in two's complement format.
For example, an `int16_t` of `-3` is stored as `0xFFFD`.

Unlike other integers in CBOR, numbers MUST NOT be stored in a variable-width encoding.
For example, a `uint16_t` always consumes 2 bytes, regardless of its value.

The length of an HNA-tagged byte string MUST be a multiple of the natural width of the numeric data type.
For example, byte string tagged with tag 1111 must have a length that is a multiple of 8 (as the natural width of a double-precision float is 8 bytes).

An HNA byte string with length 0 is valid and represents an empty array of numbers.

### Indefinite-length arrays

Indefinite-length HNAs are represented using an indefinite-length byte string.
The HNA tag MUST be applied to the start-of-array encoding byte (i.e., `0x9F`).
An HNA tag MUST NOT be applied to any of the finite-length byte strings contained in an indefinite-length HNA.

Each chunk of an indefinite-length HNA MUST have length that is a multiple of the numeric width.
In other words, it is invalid to split a number across two chunks.

### Tag placement

HNA tags MAY be applied to byte strings (CBOR Major Type 2).
HNA tags MUST NOT be applied to any other data type.

Multiple HNA tags MUST NOT be applied to the same byte string.
For example, it is invalid to tag a byte string with both 1107 and 1102.
It is also invalid to repeat the same HNA tag on a byte string, so `1105(1105(b'1234'))` is invalid.

### Decoder behavior

Decoders are not required to interpret any CBOR tags, including the HNA tags presented here.
A decoder MAY ignore all HNA tags, and simply decode HNA-tagged data as an ordinary CBOR byte string.

However, a decoder MAY detect and interpret HNA tags, and convert the byte string into an application-specific format.
For example, a decoder might convert a byte string tagged with 1102 into ECMAScript's `Uint32Array`.
A decoder MUST NOT perform this conversion on any stream that is described as invalid by this document.
For example, a decoder must not attempt to convert the stream `1111(b'01')` into an array of double-precision numbers, since the length of the byte string is invalid.

A decoder MAY detect an invalid use of HNA tags and issue a warning, error, or other diagnostic.
A decoder MAY stop processing a CBOR stream that contains an invalid HNA tag.

## Rationale

The array type in CBOR permits each item to have a distinct data type.
While flexible, this makes processing large arrays of homogenously-typed integers inefficient.

Consider, for example, an audio-processing application that needs to save an array of tens of thousands of `int16_t`-typed audio samples.
Using a CBOR array of `int16_t` values, the average length of each sample value is 2.99 bytes (assuming evenly-distributed values).
This is nearly 50% storage overhead compared to simply writing out an array of `int16_t` values (which would have exactly 2 bytes per value).

Worse, an application that consumes the CBOR data cannot quickly scan past a large array of numbers, since normal CBOR numbers are variable-length.
The decoder must read and process each array item in the CBOR stream in order to find the start of the next array item.
It's not possible for a decoder to skip through or over a CBOR array in constant time.

With the HNA tags presented here, a decoder can deliver constant-time random access to each array within a file that contains several large arrays; and also constant-time random access to each item inside a large array.
With a thoughtfully-designed data schema, it is possible to store gigabytes of data on disk, and randomly seek to any spot in the file by reading only a few kilobytes.

Note that there is no HNA tag for a `uint8_t` element type, because that is redundant with an untagged byte string.

## When to use

In many cases, CBOR's built-in array (Major Type 4) of numeric values (Major Type 0, 1, and 7) is more efficient than an HNA.
CBOR's variable-width encoding ensures that commonly-used values, like 0 and 1, can be encoded with excellent space efficiency.

For example, the array `[0, 0, 0, 0x1104294, 0]` can be encoded in only 10 bytes using CBOR's built-in types.
To encode it as an HNA of `uint32_t` would require 24 bytes (including the tag).

Furthermore, the flexibility of a CBOR array is a feature: applications may find it useful to be able to insert non-numeric items, like a Null value, into a data array.

Finally, tags deliver *optional* semantics.
Not every decoder will natively support HNA tags.
Therefore, protocols that rely on HNA tags impose some additional processing burden on applications that use HNA-unaware decoders.

Therefore, applications are encouraged to use a normal CBOR array for most data.

HNAs may be advisable when:

 * The data contains large quantities of numbers (thousands of values, or more)
 * It is useful to be able to access data in a random-access pattern
 * There are a large variety of numeric values (e.g., the data is not an array of mostly zeros)

## Examples

### int16_t

To represent an array of four `int16_t` values `[0x0001, 0x0203, 0x0405, -0x0001]` in an HNA:

    D9 0451                 # tag(1105): int16_t
       48                   # bytes(8)
          0001              # signed(0x0001)
          0203              # signed(0x0304)
          0506              # signed(0x0506)
          FFFF              # signed(-0x0001)

Compare to the encoding using the built-in `int16_t` in an array:

    84                      # array(4)
       01                   # unsigned(0x1)
       19 0203              # unsigned(0x203)
       19 0405              # unsigned(0x405)
       20                   # negative(0x0)

### single

To represent an array of 2 single-precision floating point values `[3.1415, -9]` in an HNA:

    D9 0457                # tag(1111): double
       48                  # bytes(8)
          40490E56         # float(3.1415)
          C1100000         # float(-9.0)

Since `-9` could be encoded as an integer or as a floating point value, there are various numerically-equivalent encodings.
The array could be encoded as a heterogenous array of a float and a negative integer:

    82                     # array(2)
       FA 40490E56         # float(3.1415)
       28                  # negative(8)

Or as two floats:

    82                     # array(2)
       FA 40490E56         # float(3.1415)
       FA C1100000         # float(-9.0)

### Indefinite-length

To represent a `uint16_t` HNA of indefinite length, where the first chunk contains `[0x8abc, 0xdef0]` and the second chunk contains `[0x1234]`:

    D9 044C               # tag(1100): uint16_t
        9F                # start indefinite byte array
            44            # bytes(4)
                8ABC      # unsigned(0x8abc)
                DEF0      # unsigned(0xdef0)
            42            # bytes(2)
                1234      # 0x1234
        FF                # end indefinite byte array

### Invalid encodings

The below byte stream is invalid, because the length of the byte string is not a multiple of the size of `uint16_t`:

    D9 044C               # tag(1100): uint16_t
        43                # bytes(3): invalid!
            012345        # ????

The below byte stream is invalid, because multiple HNA tags cannot be applied to the same byte string:

    D9 044C               # tag(1100): uint16_t
        D9 044D           # tag(1101): uint32_t
            44            # bytes(4)
                01234567  # ????

The below byte stream is invalid, because tags cannot be used inside an indefinite-length HNA:

    D9 044C               # tag(1100): uint16_t
        9F                # start indefinite byte array
            D9 044C       # tag(1100): invalid!
                42        # bytes(2)
                    8ABC  # ????
        FF                # end indefinite byte array

The below byte stream is invalid, because even though its total length is 2 bytes, the lengths of finite chunks are not a multiple of 2:

    D9 044C               # tag(1100): uint16_t
        9F                # start indefinite byte array
            41            # bytes(1): invalid!
                01        # ????
            41            # bytes(1): invalid!
                02        # ????
        FF                # end indefinite byte array

## Revision history

February 2018: Version 0.9

## References

[1] C. Bormann, and P. Hoffman. "Concise Binary Object Representation (CBOR)". RFC 7049, October 2013.

## Author

Jeffrey Tippet <jeffrey.tippet@outlook.com>

