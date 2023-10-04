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

For more examples, or if you want to test compatibility with another implementation, see compatibility tests in [varint_tests.txt](varint_tests.txt).

## How to use it

```
npm install --save bijective-varint
```

Then:

```javascript
const varint = require('bijective-varint')

const enc = varint.encode(2020304050)
// enc = Uint8Array(5) [ 240, 104, 75, 36, 50 ]

const dec = varint.decode(enc)
// dec = 2020304050
```

## API reference

The API is defined and exposed via the following typescript definitions:

```typescript
/**
 * Maximum number of bytes needed to store a javascript `number` up to 64 bits.
 * This is the max size of numbers when calling `encode[Into]` and `decode`.
 *
 * Implementor's note: We require allocation of 9 bytes, but this could
 * actually be set to 8 since we don't support normal numbers past
 * MAX_SAFE_INTEGER (53 bits). The largest safe integer fits in 8 bytes, not 9.
 */
const MAX_INT_LEN = 9;

/**
 * Maximum number of bytes needed to store the biggest supported bigint (128 bits).
 * This is the max size of numbers when calling `encode[Into]BN` and `decodeBN`.
 */
const MAX_BIGINT_LEN = 19;

/**
 * Assuming the start of a Uint8Array contains a varint, this method return the number of bytes
 * the varint takes up
 */
function bytesUsed(bytes: Uint8Array): number;

/**
 * This method checks to see if the given byte buffer has been filled with enough bytes
 * to contain a varint. This is useful for network protocols where messages are prefixed
 * with a length (varint), but you don't know if you've read enough bytes to contain the
 * message's length (since the length of the length is variable!)
 *
 * You could always just try and parse the next number and catch the thrown exception, but
 * its better if exceptions aren't thrown on the normal execution path in javascript.
 */
function bufContainsVarint(bytes: Uint8Array): boolean;

/**
 * Encode the given unsigned number as a varint. Returns the varint in a Uint8Array.
 *
 * This method is a wrapper around `encodeInto`. If you're encoding into
 * a buffer, its more efficient to use `encodeInto` directly to avoid
 * the unnecessary Uint8Array allocation here and the copy into the destination
 * buffer.
 *
 * NOTE: This method uses unsigned varint encoding. If you want to encode a signed
 * number, call encode(zigzagEncode(num)).
 */
function encode(num: number): Uint8Array;

/**
 * Encode the specified unsigned number into varint encoding, into the provided
 * Uint8Array at the specified offset. Returns number of bytes consumed in dest.
 * The passed array must have enough capacity for MAX_INT_LEN bytes (9 bytes).
 *
 * The number must be within the javascript safe integer range (53 bits).
 *
 * NOTE: This method only handles unsigned integers. Use zigzag encoding for signed
 * integers before passing your number into this method. Eg encodeInto(zigzagEncode(num), ..)
 **/
function encodeInto(num: number, dest: Uint8Array, offset: number): number;

/**
 * Decode the varint contained in a Uint8Array. The number is returned.
 *
 * This method might not use all the bytes of the result. Use bytesUsed() to
 * figure out how many bytes of the input were consumed by this method.
 */
function decode(bytes: Uint8Array): number;

/**
 * Encode the given bigint as a varint. Returns the encoded number in a Uint8Array.
 *
 * This method is a wrapper around `encodeIntoBN`. If you're encoding into
 * a buffer, its more efficient to use `encodeIntoBN` directly to avoid
 * the unnecessary Uint8Array allocation here and the copy into the destination
 * buffer.
 *
 * NOTE: This method uses unsigned varint encoding. If you want to encode a signed
 * number, call encodeBN(zigzagEncodeBN(num)).
 */
function encodeBN(num: bigint): Uint8Array;

/** The largest unsigned bigint we can encode (2^128 - 1) */
const MAX_SAFE_BIGINT: bigint;

/**
 * Encode the specified unsigned bigint into varint encoding, into the provided
 * Uint8Array at the specified offset. Returns number of bytes consumed in dest.
 * The passed array must have enough capacity for MAX_BIGINT_LEN bytes (19 bytes).
 *
 * NOTE: This method only handles unsigned integers. Use zigzag encoding for signed
 * integers before passing your number into this method. Eg:
 * encodeIntoBN(zigzagEncodeBN(num), ...).
 *
 * bijective-varint encoding only supports numbers up to 128 bits. This method
 * will fail (throw an exception) if you pass a number which does not fit within
 * the safe range.
 **/
function encodeIntoBN(num: bigint, dest: Uint8Array, offset: number): number;

/**
 * Decode the varint contained in a Uint8Array into a bigint. The number is
 * returned.
 *
 * This method might not use all the bytes of the result. Use bytesUsed() to
 * figure out how many bytes of the input were consumed by this method.
 *
 * Callers must ensure the entire number is ready in the buffer before calling
 * this method.
 */
function decodeBN(bytes: Uint8Array): bigint;

/** Zigzag encode a signed integer in a number into an unsigned integer */
function zigzagEncode(val: number): number;

/** Zigzag decode an unsigned integer into a signed integer */
function zigzagDecode(val: number): number;

/** Zigzag encode a signed integer in a bigint into an unsigned bigint */
function zigzagEncodeBN(val: bigint): bigint;

/** Zigzag decode an unsigned bigint into a signed bigint */
function zigzagDecodeBN(val: bigint): bigint;
```

# License

MIT licensed.