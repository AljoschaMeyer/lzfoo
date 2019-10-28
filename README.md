# LZFoo

A rather simple encoding for [LZ77](https://en.wikipedia.org/wiki/LZ77_and_LZ78)-style compressed data. Data is represented as either a literal or as a backpointer to previously decompressed data - no [entropy encoding](https://en.wikipedia.org/wiki/Entropy_encoding) involved. This leads to simple (and thus fast) decompression, at the cost of not being able to achieve compression ratios competetive with what e.g. [DEFLATE](https://en.wikipedia.org/wiki/DEFLATE) or [Zstandard](https://en.wikipedia.org/wiki/Zstandard) can achieve. LZFoo plays in the same league as [snappy](https://github.com/google/snappy) or [lz4](https://github.com/lz4/lz4). This spec merely describes an encoding for compressed data, *not* an algorithm for performing the compression.

**Status: Pending stabilization, will be stabilized on 01/04/2020, any further changes will then be released as a new format.**

## The Encoding Format

Conceptually, data is encoded as a sequence of elements, where each element is either:

- a literal bytestring of `length` up to 2^64 - 1
- a copy instruction: a non-zero `offset` into the already decompressed data, and a `length` of how many bytes to copy, both of length up to 2^64 - 1

When decoding, literals are appended to the output, and copy instructions append data that has already been decompressed to the output. The offset to the point from which copying begins is applied backwards from the end of the current output. The amount of bytes to copy can be greater than the offset. The semantics can be imagined as if each byte was copied one after the other - when the previous end of the output is reached, then the freshly copied bytes get re-copied. This leads to a sort of run-length encoding. For example, if the already decompressed data was `abc` and a copy instruction `copy 5 bytes from 2 bytes back` is encountered, the new output becomes `abcbcbcb`.

The actual encoding: A literal bytestring starts with a byte whose most-significant bit is zero (a number between 0 and 127). The way the `length` of the literal is encoded is determined by the remaining seven bits: if the number is below 120, the number itself is the `length`. Else, the initial byte is followed by some additional bytes that indicate the `length`, depending on the number:

| least significant seven bits | number of additional bytes |
|------------|----------------------------|
| 120 | 1 |
| 121 | 2 |
| 122 | 3 |
| 123 | 4 |
| 124 | 5 |
| 125 | 6 |
| 126 | 7 |
| 127 | 8 |

These additional bytes hold the big-endian representation of the `length`.

In all cases, this is followed by `length` many bytes, to be appended to the decompressed output verbatim.

A copy instruction starts with a byte whose most-significant bit is one (a number between 128 and 255). The `length` of the segment to copy is encoded just like the `length` in a literal bytestring. But instead of being followed by literal data to copy, it is followed by a [VarU64](https://github.com/AljoschaMeyer/varu64) that indicates the offset from where to start copying.

## Is This Really Needed?

Probably not. But snappy's format is fairly restrictive (can only copy at most 64 bytes with a single backreference), the LZ4 format uses a really weird varint, and both encode integers in little-endian rather than network byte order. The endianess was the main reason for defining yet another encoding, layering different protocols with different endianess criteria seemed weird.
