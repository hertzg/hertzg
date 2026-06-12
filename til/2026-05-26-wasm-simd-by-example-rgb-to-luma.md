# WASM SIMD by example: turning 16 RGB pixels into grayscale at once

Today I actually understood WASM SIMD by writing a real kernel with it: RGB → grayscale luma, 16 pixels per
instruction instead of one at a time. Notes for future me. (Full story: the dither
[blog post](../blog/2026-05-25-eink-dithering-wasm-simd-64ms-to-16ms.md).)

## The one idea behind all of it

A `v128` is just **128 bits = 16 bytes**. The *type prefix* on an op decides how you slice those 16 bytes into
"lanes". Same bits, three views:

```
i8x16:  16 lanes ×  8-bit   [b0][b1]...[b15]
i16x8:   8 lanes × 16-bit   [ h0 ][ h1 ]...[ h7 ]
i32x4:   4 lanes × 32-bit   [   w0   ]...[   w3   ]
```

"Lane-wise" = the op runs on every lane in parallel. One instruction, N results. That's the whole win.

## The problem

Grayscale (Rec.709): `L = 0.2126·R + 0.7152·G + 0.0722·B`, one output per pixel. Every pixel is independent, so it
*should* vectorize perfectly. The friction is purely mechanical: pixels arrive interleaved (`RGBRGBRGB…`) but SIMD
wants each channel contiguous, and bytes are too narrow to multiply without overflowing.

## Step 1: load 16 pixels (48 bytes) and de-interleave

Three loads cover 48 bytes = 16 RGB pixels exactly. Then `i8x16.shuffle` rearranges bytes: pick each output byte
from one of 32 input bytes (indices 0 to 15 = first arg, 16 to 31 = second):

```ts
const ld0 = v128.load(p);      // R0 G0 B0 R1 G1 B1 ... R5
const ld1 = v128.load(p + 16); // G5 B5 R6 ... R10 G10
const ld2 = v128.load(p + 32); // B10 R11 ... G15 B15

// gather all the R bytes into one plane (two shuffles, because R is spread across all three loads)
const tmpR = i8x16.shuffle(ld0, ld1, 0,3,6,9,12,15, 18,21,24,27,30, 0,0,0,0,0);
const vR   = i8x16.shuffle(tmpR, ld2, 0,1,2,3,4,5,6,7,8,9,10, 17,20,23,26,29);
// vR = [R0 R1 R2 ... R15]   ← 16 reds, contiguous. Same trick builds vG, vB.
```

`RGBRGB…` in, `RRRR… / GGGG… / BBBB…` out. This "planar" layout is what makes the rest trivial.

## Step 2: widen u8 → u16 (you can't multiply bytes)

`255 × 23436` overflows a byte (and a u16 product needs care), so widen first. `extend_low`/`extend_high` zero-extend
the low/high 8 bytes into 8 × u16:

```ts
const rLo = i16x8.extend_low_i8x16_u(vR);  // pixels 0..7  as u16
const rHi = i16x8.extend_high_i8x16_u(vR); // pixels 8..15 as u16
// _u = unsigned (zero-extend), _s would sign-extend
```

## Step 3: multiply + accumulate in Q15 fixed-point

The float coefficients become integers: `round(coef × 2^15)`. They sum to exactly 32768, so a single `>>15` at the
end divides it back out.

```ts
const cR = i16x8.splat(6966);   // 0.2126 × 32768   (splat = broadcast to all lanes)
const cG = i16x8.splat(23436);  // 0.7152 × 32768
const cB = i16x8.splat(2366);   // 0.0722 × 32768

// extmul = "extending multiply": 8×u16 × 8×u16 → 4×i32 products, no overflow.
let acc = i32x4.extmul_low_i16x8_u(rLo, cR);          // R contribution, pixels 0..3
acc = i32x4.add(acc, i32x4.extmul_low_i16x8_u(gLo, cG)); // + G
acc = i32x4.add(acc, i32x4.extmul_low_i16x8_u(bLo, cB)); // + B
```

Key point: sum R+G+B in **i32 before rounding**, so there's exactly one rounding error per pixel.

## Step 4: round, shift, narrow back down, store

```ts
acc = i32x4.shr_s(i32x4.add(acc, i32x4.splat(0x4000)), 15); // +½, then ÷32768 → luma 0..255
// pack two i32x4 (8 values) back into one i16x8 and write 8 results at once
v128.store(out, i16x8.narrow_i32x4_s(acc, acc1));
```

## The mental model that finally stuck

Every SIMD kernel is the same four moves: **load → rearrange (shuffle) → widen → multiply/add → narrow → store.**
Most of the "weird" ops (shuffle, extend, narrow) exist only because lanes are fixed-width, they're plumbing to get
data into and out of the width the math needs. The actual arithmetic is the easy part.

Gotchas I hit:
- **`--enable simd`** has to be passed to `asc`, or `v128` doesn't compile.
- **Lane width must match the op.** `i32x4.add` on data you loaded as bytes will silently treat 4 bytes as one int.
- A **scalar tail** is still needed for `width % 16` pixels; for my fixed panel widths (800/1872/3840) it never runs,
  but leave it in or you'll corrupt the last chunk.
- Going integer (Q15) made the kernel **bit-deterministic**, the float version drifted ±1 vs the reference across
  platforms, the integer one doesn't.
