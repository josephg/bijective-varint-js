# Bijective varint encoding

This is a simple, zero dependency javascript library which implements length-prefixed varint encoding. The format is similar to LEB128 but:

- It uses a bijective base, based off [this HN comment](https://news.ycombinator.com/item?id=11263378). This means every number has exactly 1 canonical binary encoding.
- The length prefixed encoding should play better with the branch predictor in native implementations. (Though this implementation isn't super optimized).

This format is extremely similar to how UTF8 works internally. Its almost certainly possible to reuse existing efficient UTF8 <-> UTF32 SIMD encoders and decoders to make this code faster, but frankly its not a priority right now.

- `0    - 2^7-1` encodes as `0b0xxx_xxxx`
- `2^7  - 2^14+2^7-1` encodes as `0b10xx_xxxx xxxx_xxxx`
- `2^14+2^7 - 2^21+2^14+2^7-1` encodes as `0b110x_xxxx xxxx_xxxx xxxx_xxxx`
- `2^21 - 2^28-1` encodes as `0b1110_xxxx xxxx_xxxx xxxx_xxxx xxxx_xxxx`

... And so on.

The encoding supports any integer up to 128 bits.

For numbers above 64 bits, it would be tempting to use `0x1111_1111 1111_1111 xxxx_xxxx xxxx_xxxx ...` since then there would be at most 2 bytes of overhead (or 4 bytes of overhead for 128 bits). But that breaks the pattern, so instead it uses this as the maximum encoding for 64 bits: `0x1111_1111 1111_1111 0xxx_xxxx` ... And for 128 bits: `0x1111_1111 1111_1111 1111_1111 1111_1111 0xxx_xxxx`.

For negative integers, we use protobuf's [zigzag encoding](https://protobuf.dev/programming-guides/encoding/) format. (0 encodes as 0, -1 encodes as 1, 1 encodes as 2, -2 encodes as 3, and so on).
