# wasm — Kernels

The eleven kernels that ship in the current main. Every one is a `go run`
command that prints WAT to stdout; every one has a golden-file test that
pins the exact output byte-for-byte; every one has a wazero cross-check
against a Go reference.

## matchlen — `bytes.Equal`-shaped scan

Signature:

```lisp
(func $matchlen16 (param $ptrA i32) (param $ptrB i32) (param $limit i32)
                  (result i32))
```

Returns the number of leading bytes at `ptrA` that match the corresponding
bytes at `ptrB`, up to `limit`. Two-loop shape: a 16-byte SIMD fast loop for
whole-chunk equality, then a scalar tail that finds the exact first-mismatch
offset within the byte after the fast loop bails.

Emit-surface ops it exercises: `v128.load`, `i8x16.eq`, `i8x16.all_true`,
`i32.load8_u`, and the control-flow primitives `block` / `loop` / `br_if`.

## hex — `encoding/hex`-shaped encoder

Signature:

```lisp
(func $hex_encode (param $dstPtr i32) (param $srcPtr i32) (param $nBlocks i32))
```

Converts each 16-byte input chunk to 32 hex chars, matching Go's
`encoding/hex.EncodeToString` byte-for-byte. Uses the classic SSSE3 recipe:

- Split each byte into its low and high nibbles with `i8x16.shr_u` + a
  low-nibble mask.
- Look up each nibble in a 16-entry LUT via `i8x16.swizzle` — the wasm-SIMD
  analogue of PSHUFB.
- Interleave the two chars per byte via a fixed `i8x16.shuffle` pattern.

## popcount — `math/bits.OnesCount`-shaped reduction

Signature:

```lisp
(func $popcount (param $srcPtr i32) (param $nBlocks i32) (result i32))
```

Counts set bits across `nBlocks × 16` input bytes. This is the kernel where
wasm-SIMD `i8x16.popcnt` earns its keep — SSE has popcount only on scalar
GPR64, and NEON's `CNT` is a nibble-level count. Wasm-SIMD's per-byte popcnt
plus the pairwise-widening reduction chain
(`i16x8.extadd_pairwise_i8x16_u` → `i32x4.extadd_pairwise_i16x8_u` →
`i32x4.add`) fits a whole-buffer popcount in ~5 v128 ops per 16 input bytes,
with the final horizontal sum via four `i32x4.extract_lane` + three
`i32.add`.

## toupper — `strings.ToUpper` for ASCII

Signature:

```lisp
(func $toupper (param $dstPtr i32) (param $srcPtr i32) (param $nBlocks i32))
```

Case-folds ASCII bytes to uppercase 16 lanes at a time. Every byte outside
`[a..z]` passes through untouched. The kernel showcases the pattern common to
text-processing kernels: a **signed-compare pair → mask → arithmetic delta**:

```text
mask   = (bytes ge_s 'a') & (bytes le_s 'z')     ; 0xff where in [a..z]
delta  = mask & (0x20 × 16)                       ; 0x20 where lowercase
upper  = i8x16.sub(bytes, delta)                  ; branchless subtract
```

## memchr — `bytes.IndexByte`-shaped search

Signature:

```lisp
(func $memchr (param $srcPtr i32) (param $nBlocks i32) (param $needle i32)
              (result i32))
```

Returns the index of the first byte in the scanned region that equals
`needle`, or `-1` if none match. The kernel is where `i8x16.bitmask` earns
its keep — SSE has PMOVMSKB with the same shape (MSB of each lane compressed
into a scalar bitmask), and wasm-SIMD gives us the same construction:

```text
key    = i8x16.splat(needle)         ; broadcast search byte
bytes  = v128.load(src + 16*i)
mask   = i8x16.bitmask(bytes == key) ; MSB-of-lane → i32 (0..0xffff)
if mask != 0:
  return i*16 + i32.ctz(mask)
```

## isascii — `all(b < 128 for b in buf)` preflight

Signature:

```lisp
(func $isascii (param $srcPtr i32) (param $nBlocks i32) (result i32))
```

Returns 1 iff every scanned byte is in the 7-bit ASCII range, else 0.
Every parser (JSON, HTTP, URL, CSV, XML, protobuf-JSON, …) preflights this
before choosing between a fast ASCII path and a slower multibyte-aware
path. The kernel is compact because wasm-SIMD gives it the tightest shape
possible: `i8x16.bitmask` interprets each lane's MSB as the sign bit, and
for a byte value the sign bit and the "is >= 128" bit are the same bit.

```text
bytes = v128.load(src + 16*i)
if i8x16.bitmask(bytes) != 0:
  return 0     ; saw a byte >= 128 in this block
; else advance i
```

No extra compare, no key vector, no shift — just load, bitmask, i32.eqz.

## hex_decode — `encoding/hex.DecodeString`-shape (companion to `hex`)

Signature:

```lisp
(func $hex_decode (param $dstPtr i32) (param $srcPtr i32) (param $nBlocks i32))
```

Each block consumes 16 hex characters and writes 8 raw bytes — the exact
inverse of the encoder's 16-byte → 32-char expansion. The kernel showcases
the "three-range arithmetic offset" pattern that recurs in every ASCII-to-
value conversion (base16, base32, base64, url-percent decoding):

```text
digit    = (chars >= '0') & (chars <= '9')                  ; 0xff on 0-9
upper    = (chars >= 'A') & (chars <= 'F')                  ; 0xff on A-F
lower    = (chars >= 'a') & (chars <= 'f')                  ; 0xff on a-f
offset   = (digit & 48) | (upper & 55) | (lower & 87)       ; per-lane
nibbles  = chars - offset                                    ; each lane 0..15
```

The three offsets encode the ASCII → nibble arithmetic: `'0'-0 = 48` for
digits, `'A'-10 = 55` for uppercase, `'a'-10 = 87` for lowercase. AND-with-
mask + OR combines them into a single per-lane offset vector so each lane
subtracts exactly the offset for its own range.

Pack: gather even-indexed nibbles into the low 8 lanes via `i8x16.shuffle`,
shift left 4, OR with the odd-indexed nibbles gathered the same way, and
store just the low 8 bytes via `v128.store64_lane offset=0 0`. New emit-
surface ops: `i8x16.shl` (per-lane left shift, symmetric to `i8x16.shr_u`)
and `v128.store64_lane` (partial store of one i64 lane of a v128).

## utf8len — `utf8.RuneCount` for valid UTF-8

Signature:

```lisp
(func $utf8len (param $srcPtr i32) (param $nBlocks i32) (result i32))
```

Counts runes (Unicode code points) in a scanned buffer of valid UTF-8
bytes. The technique is a two-step horizontal reduction that mirrors
popcount at a coarser grain: continuation bytes have the exact `10xxxxxx`
pattern, so ANDing the input with `0xc0×16` and comparing to `0x80×16`
gives `0xff` exactly on continuation lanes. `i8x16.bitmask` compresses
those 16 truth bits to a scalar i32, and the new `i32.popcnt` op counts
how many of the 16 lanes were continuations. The rune count is the
number of non-continuation bytes: `16 - popcnt(mask)`.

```text
bytes = v128.load(src + 16*i)
bit10 = i8x16.eq( bytes & (0xc0 × 16), 0x80 × 16 )   ; 0xff on cont
cont  = i32.popcnt( i8x16.bitmask(bit10) )           ; count in block
count += 16 - cont                                     ; rune starts
```

This is exact on valid UTF-8 — every rune has exactly one non-continuation
leader byte, so the count of leaders equals the count of runes. Malformed
inputs give an approximation (the count of bytes whose top two bits aren't
`10`).

## json_clean — JSON-string bulk-copy preflight

Signature:

```lisp
(func $json_clean (param $srcPtr i32) (param $nBlocks i32) (result i32))
```

Returns 1 iff every byte in the scanned buffer can be copied verbatim
into the current JSON string body — no `"` (0x22) closing the string,
no `\` (0x5c) starting an escape, no control character (0x00..0x1f)
that would need `\uXXXX` escaping. A parser can then take the fast
`memcpy`-style bulk-copy path when this returns 1 and drop to the
byte-by-byte slow path on 0. This is the hottest inner check in almost
every JSON string-parsing loop.

```text
bytes    = v128.load(src + 16*i)
quotes   = i8x16.eq(bytes, 0x22 × 16)                ; 0xff on `"`
slashes  = i8x16.eq(bytes, 0x5c × 16)                ; 0xff on `\`
controls = i8x16.eq(v128.and(bytes, 0xe0 × 16), 0)   ; 0xff on 0x00..0x1f
dirty    = quotes | slashes | controls
if i8x16.bitmask(dirty) != 0:
  return 0
```

The control-char detection uses `(bytes & 0xe0) == 0` instead of the
obvious `i8x16.le_s(bytes, 0x1f)` because signed less-than would
misclassify high-bit bytes (0x80..0xff, negative under i8) as controls
— the wazero verifier caught this trap on the first implementation.
`bytes & 0xe0 == 0` is true iff the top three bits are all clear, which
happens exactly for byte values 0x00..0x1f regardless of signedness.

## adler32 — `hash/adler32.Checksum` (Adler-32 rolling sum)

Signature:

```lisp
(func $adler32 (param $srcPtr i32) (param $nBlocks i32)
               (param $aInit i32) (param $bInit i32)
               (param $aOut i32) (param $bOut i32))
```

Updates the two running u32 sums `a` and `b` from initial values,
processing `nBlocks × 16` bytes without any modulo (caller batches
NMAX=5552-byte segments and reduces between calls). The kernel writes
the final `a` and `b` back through caller-provided pointers so the
caller can control when to fold in mod-65521.

Per 16-byte block:

```text
bytes  = v128.load(src + 16*i)
; delta_a = sum(bytes) via the popcount-style pairwise-widen reduction
pair16 = i16x8.extadd_pairwise_i8x16_u(bytes)
pair32 = i32x4.extadd_pairwise_i16x8_u(pair16)
; delta_b_weighted = sum(bytes[k] * (16-k)) using widening extmul
coef     = [16, 15, 14, ..., 1]
prod_lo  = i16x8.extmul_low_i8x16_u(bytes, coef)
prod_hi  = i16x8.extmul_high_i8x16_u(bytes, coef)
sum_prod = i32x4.add(extadd_pairwise(prod_lo), extadd_pairwise(prod_hi))
; scalar update
b = b + 16*a + horizontal_sum(sum_prod)
a = a + horizontal_sum(pair32)
```

The `i16x8.extmul_low/high_i8x16_u` ops widen and multiply pairwise in
one instruction — no PMULLW+PMULHW dance. Max product 255×16=4080 fits
in u16 so the intermediate representation stays exact.

## base64_encode — `encoding/base64.StdEncoding` (Lemire's algorithm)

Signature:

```lisp
(func $base64_encode (param $dstPtr i32) (param $srcPtr i32)
                     (param $nBlocks i32))
```

12 input bytes → 16 output ASCII chars per iteration, matching
StdEncoding. The algorithm is Lemire's SSE encoder
(arXiv:1704.00605) adapted for wasm-SIMD, which lacks PMULHUW.

Per 12-byte block:

1. Load 16 bytes (12 real + 4 tail; tail bytes shuffle into ignored
   positions).
2. Byte-shuffle so each 3-byte triplet maps to two big-endian u16
   lanes `(b0<<8|b1)` and `(b1<<8|b2)` — 24 bits spread across two
   adjacent i16 words with 4 bits of overlap.
3. Extract the four 6-bit indexes per triplet in two rounds:
    - **Round 1 (PMULHUW-style)**: mask `0x0fc0_fc00`, multiply by
      `0x0400_0040`. Wasm-SIMD lacks PMULHUW, so this is emulated with
      `i32x4.extmul_low/high_i16x8_u` + `i32x4.shr_u` by 16 +
      `i16x8.narrow_i32x4_u`.
    - **Round 2 (PMULLW-style)**: mask `0x003f_03f0`, multiply by
      `0x0100_0010` via `i16x8.mul` (which IS PMULLW).
4. OR the two rounds — the v128 now holds all 16 6-bit indexes
   sequentially in its byte lanes.
5. Range-add ASCII conversion (each conditional is a mask
   AND-ed with the offset, summed via `i8x16.add`):

    ```text
    result = idx + 65
             + (idx > 25 ? +6   : 0)   ; 'a'..'z' offset
             + (idx > 51 ? -75  : 0)   ; '0'..'9' offset
             + (idx == 62 ? -15 : 0)   ; '+'
             + (idx == 63 ? -12 : 0)   ; '/'
    ```

    Negative offsets go modular (-75 → 181, -15 → 241, -12 → 244).
    The wazero verifier caught a real bug on the `/` correction
    during development: an initial -16 (240) landed idx=63 at '+'
    instead of '/'. The correct value is -12 (244).

## The emit surface, catalogued

Every op the eleven kernels reach into is a one-line method on `Function`.
Extension is by declarative append — see
[`emit.go`](https://github.com/go-asmgen/asmgen/blob/main/wasm/emit.go).

| Family | Ops |
|---|---|
| Memory | `v128.load`, `v128.store`, `v128.store64_lane`, `i32.load8_u` |
| Locals + stack | `local.get/set/tee`, `i32.const` |
| v128 boolean | `v128.and`, `v128.or`, `v128.xor`, `v128.zero` |
| i8x16 compare / arith | `i8x16.eq`, `i8x16.ge_s`, `i8x16.le_s`, `i8x16.all_true`, `i8x16.sub`, `i8x16.shr_u`, `i8x16.shl`, `i8x16.popcnt`, `i8x16.splat` |
| Horizontal aggregation | `i8x16.bitmask` |
| Shuffle / swizzle | `i8x16.swizzle`, `i8x16.shuffle`, `v128.const` |
| Widening reductions | `i16x8.extadd_pairwise_i8x16_u`, `i32x4.extadd_pairwise_i16x8_u`, `i32x4.add`, `i32x4.extract_lane` |
| Scalar | `i32.add`, `i32.sub`, `i32.mul`, `i32.ctz`, `i32.popcnt`, `i32.eqz`, `i32.gt_s`, `i32.ge_s`, `i32.ne` |
| Control flow | `block`, `loop`, `br_if`, `br`, `return` |
