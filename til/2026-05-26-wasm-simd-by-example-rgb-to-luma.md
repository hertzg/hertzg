# WASM SIMD by example: turning 16 RGB pixels into grayscale at once

Today I actually understood WASM SIMD by writing a real kernel with it: RGB → grayscale luma, 16 pixels per
instruction instead of one at a time. Notes for future me. (Full story: the dither
[blog post](../blog/2026-05-25-eink-dithering-wasm-simd-64ms-to-16ms.md).)

Spoiler first, theory after. The entire trick of this page: take code A, a loop so innocent it could be a
job-interview warmup, teach it to speak 16-pixels-at-a-time lane language (code B), and the same math comes out
**~4× faster**. One file, nothing to install besides deno, and it compiles its own AssemblyScript at startup:

<details markdown="1">
<summary>🪄 code A → code B, runnable and benchmarked (click to spoil yourself)</summary>

```ts
// luma_bench.ts, fully self-contained: the AssemblyScript compiles itself on the fly.
// Run: deno bench -A luma_bench.ts
import asc from "npm:assemblyscript/asc";

// ---------- code A: the loop anyone would write ----------

function rgbToLumaNaive(rgb: Uint8Array, out: Uint8Array): void {
  const pixelCount = out.length;
  for (let i = 0; i < pixelCount; i++) {
    const r = rgb[i * 3];
    const g = rgb[i * 3 + 1];
    const b = rgb[i * 3 + 2];
    out[i] = Math.round(0.2126 * r + 0.7152 * g + 0.0722 * b);
  }
}

// ---------- code A.5: same loop, float math swapped for Q15 integers ----------

function rgbToLumaScalar(rgb: Uint8Array, out: Uint8Array): void {
  const pixelCount = out.length;
  for (let i = 0; i < pixelCount; i++) {
    const r = rgb[i * 3];
    const g = rgb[i * 3 + 1];
    const b = rgb[i * 3 + 2];
    out[i] = (6966 * r + 23436 * g + 2366 * b + 0x4000) >> 15;
  }
}

// ---------- code B: the same math, spoken in 16-wide lane language ----------

const source = `
export function luma(inPtr: usize, outPtr: usize, pixels: i32): void {
  const redWeight = i16x8.splat(6966);  // 0.2126 in Q15
  const greenWeight = i16x8.splat(23436); // 0.7152 in Q15
  const blueWeight = i16x8.splat(2366);  // 0.0722 in Q15
  const q15Half = i32x4.splat(0x4000);

  for (let i = 0; i + 16 <= pixels; i += 16) {
    const chunkPtr = inPtr + <usize>(i * 3);

    // load 48 interleaved bytes, de-interleave into R/G/B planes
    const chunk0 = v128.load(chunkPtr);
    const chunk1 = v128.load(chunkPtr, 16);
    const chunk2 = v128.load(chunkPtr, 32);
    const redPartial = i8x16.shuffle(chunk0, chunk1, 0,3,6,9,12,15,18,21,24,27,30,0,0,0,0,0);
    const redPlane = i8x16.shuffle(redPartial, chunk2, 0,1,2,3,4,5,6,7,8,9,10,17,20,23,26,29);
    const greenPartial = i8x16.shuffle(chunk0, chunk1, 1,4,7,10,13,16,19,22,25,28,31,0,0,0,0,0);
    const greenPlane = i8x16.shuffle(greenPartial, chunk2, 0,1,2,3,4,5,6,7,8,9,10,18,21,24,27,30);
    const bluePartial = i8x16.shuffle(chunk0, chunk1, 2,5,8,11,14,17,20,23,26,29,0,0,0,0,0,0);
    const bluePlane = i8x16.shuffle(bluePartial, chunk2, 0,1,2,3,4,5,6,7,8,9,16,19,22,25,28,31);

    // widen u8 -> u16
    const redLow = i16x8.extend_low_i8x16_u(redPlane);
    const redHigh = i16x8.extend_high_i8x16_u(redPlane);
    const greenLow = i16x8.extend_low_i8x16_u(greenPlane);
    const greenHigh = i16x8.extend_high_i8x16_u(greenPlane);
    const blueLow = i16x8.extend_low_i8x16_u(bluePlane);
    const blueHigh = i16x8.extend_high_i8x16_u(bluePlane);

    // multiply + accumulate in Q15, 4 pixels per accumulator
    let luma0to3 = i32x4.extmul_low_i16x8_u(redLow, redWeight);
    luma0to3 = i32x4.add(luma0to3, i32x4.extmul_low_i16x8_u(greenLow, greenWeight));
    luma0to3 = i32x4.add(luma0to3, i32x4.extmul_low_i16x8_u(blueLow, blueWeight));
    let luma4to7 = i32x4.extmul_high_i16x8_u(redLow, redWeight);
    luma4to7 = i32x4.add(luma4to7, i32x4.extmul_high_i16x8_u(greenLow, greenWeight));
    luma4to7 = i32x4.add(luma4to7, i32x4.extmul_high_i16x8_u(blueLow, blueWeight));
    let luma8to11 = i32x4.extmul_low_i16x8_u(redHigh, redWeight);
    luma8to11 = i32x4.add(luma8to11, i32x4.extmul_low_i16x8_u(greenHigh, greenWeight));
    luma8to11 = i32x4.add(luma8to11, i32x4.extmul_low_i16x8_u(blueHigh, blueWeight));
    let luma12to15 = i32x4.extmul_high_i16x8_u(redHigh, redWeight);
    luma12to15 = i32x4.add(luma12to15, i32x4.extmul_high_i16x8_u(greenHigh, greenWeight));
    luma12to15 = i32x4.add(luma12to15, i32x4.extmul_high_i16x8_u(blueHigh, blueWeight));

    // round, shift out of Q15, narrow i32 -> i16 -> u8, store 16 results at once
    luma0to3 = i32x4.shr_s(i32x4.add(luma0to3, q15Half), 15);
    luma4to7 = i32x4.shr_s(i32x4.add(luma4to7, q15Half), 15);
    luma8to11 = i32x4.shr_s(i32x4.add(luma8to11, q15Half), 15);
    luma12to15 = i32x4.shr_s(i32x4.add(luma12to15, q15Half), 15);
    const luma0to7 = i16x8.narrow_i32x4_s(luma0to3, luma4to7);
    const luma8to15 = i16x8.narrow_i32x4_s(luma8to11, luma12to15);
    v128.store(outPtr + <usize>i, i8x16.narrow_i16x8_u(luma0to7, luma8to15));
  }
}
`;

// compile in memory and import the bytes as a module, no files involved
const { error, binary, stderr } = await asc.compileString(source, {
  optimizeLevel: 3,
  runtime: "stub",
  enable: ["simd"],
});
if (error) throw new Error(stderr.toString());

const { memory, luma } = await import(
  `data:application/wasm;base64,${binary!.toBase64()}`
) as { memory: WebAssembly.Memory; luma: (i: number, o: number, n: number) => void };

// ---------- one 12-megapixel photo (4000x3000) ----------

const N = 4000 * 3000;
const IN = 65536, OUT = IN + N * 3;
memory.grow(Math.ceil((OUT + N) / 65536)); // stub runtime starts at 0 pages

const rgbWasm = new Uint8Array(memory.buffer, IN, N * 3); // views AFTER grow (grow detaches)
for (let i = 0; i < rgbWasm.length; i++) rgbWasm[i] = (i * 2654435761) >>> 24;

const rgbJs = rgbWasm.slice(); // the JS versions get their own plain copy, fair is fair
const outJs = new Uint8Array(N);

// ---------- the receipts ----------

Deno.bench("code A: naive float", () => {
  rgbToLumaNaive(rgbJs, outJs);
});

Deno.bench("code A.5: scalar Q15", () => {
  rgbToLumaScalar(rgbJs, outJs);
});

Deno.bench("code B: WASM SIMD", () => {
  luma(IN, OUT, N);
});
```

And the receipts, M2 Pro:

```
$ deno bench -A luma_bench.ts
| benchmark              | time/iter (avg) |  iter/s |
| ---------------------- | --------------- | ------- |
| code A: naive float    |         23.3 ms |    42.8 |
| code A.5: scalar Q15   |         19.1 ms |    52.3 |
| code B: WASM SIMD      |          5.7 ms |   174.1 |
```

~4× on a single core for a 12-megapixel photo (and the integer trick alone is worth ~20% before any SIMD).
Every weird-looking line in code B is explained below.

> **Note:**
> The SIMD bench reads and writes buffers that already live inside wasm memory, while the JS versions get plain
> arrays — fair for measuring the kernel itself, but in real use the pixels have to get into wasm memory first.
> The [dither post](../blog/2026-05-25-eink-dithering-wasm-simd-64ms-to-16ms.md) covers that plumbing.

</details>

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

Grayscale (Rec.709): `L = 0.2126·R + 0.7152·G + 0.0722·B`, one output per pixel. The textbook version, exactly as
naive as it sounds:

```ts
function rgbToLuma(rgb: Uint8Array): Uint8Array {
  const pixelCount = rgb.length / 3;       // input is interleaved: R G B R G B ...
  const luma = new Uint8Array(pixelCount); // output: one 0..255 gray byte per pixel

  for (let i = 0; i < pixelCount; i++) {
    const r = rgb[i * 3];
    const g = rgb[i * 3 + 1];
    const b = rgb[i * 3 + 2];
    luma[i] = Math.round(0.2126 * r + 0.7152 * g + 0.0722 * b);
  }
  return luma;
}
```

Every pixel is independent, so it *should* vectorize perfectly. The friction is purely mechanical: pixels arrive
interleaved (`RGBRGBRGB…`) but SIMD wants each channel contiguous, and bytes are too narrow to multiply without
overflowing.

## The scalar version, for contrast

Same loop as the naive version, but with the float math swapped for integer "Q15" fixed-point, so its output is
bit-identical to the SIMD kernel — same bits out, one pixel at a time.

Q15 is a way to store fractions in plain integers: scale the fraction up by 2^15 = 32768, keep the integer, and
remember the decimal point lives 15 bits from the right. The *bits* say 6966, the *meaning* is 6966 / 32768 ≈ 0.2126:

```
0.2126  →  round(0.2126 × 32768)  =   6966
0.7152  →  round(0.7152 × 32768)  =  23436
0.0722  →  round(0.0722 × 32768)  =   2366
                                     ─────
                                     32768  = 2^15 exactly
```

Where 32768 itself comes from: 2^15, aka `1 << 15`, one factor of 2 per reserved fraction bit. It's the binary
version of "store dollars as cents": cents scale by 10^2 because you reserved 2 decimal places, Q15 scales by 2^15
because you reserved 15 binary ones.

Multiplying by a Q15 constant is integer-multiply then `>> 15`. Rounding is adding "0.5 in Q15" (`0x4000` = 2^14)
before the shift. And the constants summing to *exactly* 32768 is deliberate: dividing by the total weight becomes
an exact `>> 15`, so pure white (255,255,255) comes out as exactly 255, no bias.

> **Note:**
> For the careful reader: JS bitwise ops (`>>`, `<<`) secretly run on 32-bit signed ints, so they misbehave
> near 2^31. Everything here peaks at ~23 bits, far below the cliff, so that quirk is ignored for brevity. Read
> the integer math as if numbers had no width limit, BigInt-style, and you'll predict every result correctly.

```ts
function lumaScalar(rgb: Uint8Array, out: Uint8Array, n: number): void {
  for (let i = 0, p = 0; i < n; i++, p += 3) {
    const r = rgb[p];
    const g = rgb[p + 1];
    const b = rgb[p + 2];
    out[i] = (6966 * r + 23436 * g + 2366 * b + 0x4000) >> 15;
  }
}
```

Eight lines, completely clear, and it's what I'd write every time speed doesn't matter. Per pixel it does 3 loads,
3 multiplies, 3 adds, a shift and a store, plus the loop bookkeeping, and V8 won't auto-vectorize it. The SIMD
version below spends ~the same number of *instructions* to retire **16 pixels**, and every step exists to feed
that wide datapath: gather the bytes into planes, widen them so the math fits, multiply all lanes at once, narrow
back. Same arithmetic, different plumbing.

## Step 1: load 16 pixels (48 bytes) and de-interleave

Three loads cover 48 bytes = 16 RGB pixels exactly. Then `i8x16.shuffle` rearranges bytes: pick each output byte
from one of 32 input bytes (indices 0 to 15 = first arg, 16 to 31 = second):

```ts
const chunk0 = v128.load(chunkPtr);     // R0 G0 B0 R1 G1 B1 ... R5
const chunk1 = v128.load(chunkPtr, 16); // G5 B5 R6 ... R10 G10   (2nd arg = constant offset,
const chunk2 = v128.load(chunkPtr, 32); // B10 R11 ... G15 B15     baked into the instruction)

// gather all the R bytes into one plane (two shuffles, because R is spread across all three loads)
const redPartial = i8x16.shuffle(chunk0, chunk1, 0,3,6,9,12,15, 18,21,24,27,30, 0,0,0,0,0);
const redPlane   = i8x16.shuffle(redPartial, chunk2, 0,1,2,3,4,5,6,7,8,9,10, 17,20,23,26,29);
// redPlane = [R0 R1 R2 ... R15]   ← 16 reds, contiguous. Same trick builds greenPlane, bluePlane.
```

What `i8x16.shuffle` does, as TS (this and every "fake TS" below runs as a loop and allocates; the real op is
**one instruction**, no loop, no allocation):

```ts
function i8x16_shuffle(a: Uint8Array, b: Uint8Array, indices: number[]): Uint8Array {
  const candidates = [...a, ...b]; // 32 bytes to pick from: a = 0..15, b = 16..31

  const out = new Uint8Array(16);  // i8 → Uint8Array,  x16 → (16)
  for (let lane = 0; lane < 16; lane++) {
    out[lane] = candidates[indices[lane]]; // each output byte grabs one candidate
  }
  return out;
}
```

In pictures, the R-plane gather is just "every 3rd byte":

```
memory, 48 bytes = 16 px:   R0 G0 B0 R1 G1 B1 R2 G2 B2 ... R15 G15 B15
                            └─chunk0 (0..15)─┘└─chunk1 (16..31)─┘└─chunk2 (32..47)─┘

shuffle indices:             0  3  6  9  12 15 18 21 24 27 30 ...
                             │  │  │  │   (every 3rd byte = the R's)

redPlane   (i8x16): [R0|R1|R2|R3|R4|R5|R6|R7|R8|R9|R10|R11|R12|R13|R14|R15]
greenPlane (i8x16): [G0|G1|G2|G3|G4|G5|G6|G7|G8|G9|G10|G11|G12|G13|G14|G15]
bluePlane  (i8x16): [B0|B1|B2|B3|B4|B5|B6|B7|B8|B9|B10|B11|B12|B13|B14|B15]
```

`RGBRGB…` in, `RRRR… / GGGG… / BBBB…` out. This "planar" layout is what makes the rest trivial.

## Step 2: widen u8 → u16 (you can't multiply bytes)

`255 × 23436` overflows a byte (and a u16 product needs care), so widen first. `extend_low`/`extend_high` zero-extend
the low/high 8 bytes into 8 × u16:

```ts
const redLow = i16x8.extend_low_i8x16_u(redPlane);  // pixels 0..7  as u16
const redHigh = i16x8.extend_high_i8x16_u(redPlane); // pixels 8..15 as u16
// _u = unsigned (zero-extend), _s would sign-extend
```

As TS:

```ts
// name decode: i16x8 = result shape, extend_low = which half, i8x16 = input shape, _u = unsigned
function i16x8_extend_low_i8x16_u(v: Uint8Array): Uint16Array {
  //                            i8x16 → Uint8Array in   i16x8 → Uint16Array out
  const out = new Uint16Array(8); // i16 → Uint16Array,  x8 → (8)
  for (let lane = 0; lane < 8; lane++) {
    out[lane] = v[lane]; // bytes 0..7 ("low" half), each copied into a roomier 16-bit slot
  }
  return out;
}

function i16x8_extend_high_i8x16_u(v: Uint8Array): Uint16Array {
  const out = new Uint16Array(8);
  for (let lane = 0; lane < 8; lane++) {
    out[lane] = v[lane + 8]; // same, for bytes 8..15 (the "high" half)
  }
  return out;
}
```

```
redPlane (i8x16):  [R0|R1|R2|R3|R4|R5|R6|R7|R8|R9|R10|R11|R12|R13|R14|R15]
                    └── extend_low ──────┘└── extend_high ────────────┘
redLow   (i16x8):  [ R0  | R1  | R2  | R3  | R4  | R5  | R6  | R7  ]   each byte now
redHigh  (i16x8):  [ R8  | R9  | R10 | R11 | R12 | R13 | R14 | R15 ]   has 16 bits of room
```

## Step 3: multiply + accumulate in Q15 fixed-point

The Q15 constants from the scalar section come back, now one per lane. Why integer lanes instead of float SIMD
(`f32x4` exists):

- a pixel (≤255) times a Q15 constant (≤32768) is ~23 bits, so the i32 lanes from `extmul` can never overflow
- twice the lanes: 8 × i16 per `v128` vs 4 × f32, so half the multiply instructions
- integer math is **bit-deterministic** across platforms; floats round differently (FMA fusing etc.), integers don't.
  This is what makes the kernel bit-exact against its TS reference instead of "within ±1".

```ts
const redWeight = i16x8.splat(6966);   // 0.2126 × 32768   (splat = broadcast to all lanes)
const greenWeight = i16x8.splat(23436);  // 0.7152 × 32768
const blueWeight = i16x8.splat(2366);   // 0.0722 × 32768

// extmul = "extending multiply": 8×u16 × 8×u16 → 4×i32 products, no overflow.
let luma0to3 = i32x4.extmul_low_i16x8_u(redLow, redWeight);          // R contribution, pixels 0..3
luma0to3 = i32x4.add(luma0to3, i32x4.extmul_low_i16x8_u(greenLow, greenWeight)); // + G
luma0to3 = i32x4.add(luma0to3, i32x4.extmul_low_i16x8_u(blueLow, blueWeight)); // + B
```

As TS:

```ts
function i16x8_splat(value: number): Uint16Array {
  return new Uint16Array(8).fill(value); // i16 → Uint16Array,  x8 → (8),  splat → fill
}

// i32x4 = result (4 wide lanes), extmul = extending multiply, i16x8 = inputs, _u = unsigned
function i32x4_extmul_low_i16x8_u(a: Uint16Array, b: Uint16Array): Int32Array {
  const out = new Int32Array(4); // i32 → Int32Array,  x4 → (4)
  for (let lane = 0; lane < 4; lane++) {
    out[lane] = a[lane] * b[lane]; // input lanes 0..3 ("low"); each product gets a full 32-bit slot
  }
  return out;
}

function i32x4_add(a: Int32Array, b: Int32Array): Int32Array {
  const out = new Int32Array(4); // i32 → Int32Array,  x4 → (4)
  for (let lane = 0; lane < 4; lane++) {
    out[lane] = a[lane] + b[lane]; // lane-wise; nothing carries between lanes
  }
  return out;
}
```

```
redLow    (i16x8):  [ R0  | R1  | R2  | R3  | R4  | R5  | R6  | R7  ]
redWeight (i16x8):  [6966 |6966 |6966 |6966 |6966 |6966 |6966 |6966 ]
                     └── extmul_low (lanes 0..3) ──┘└── extmul_high (4..7) ──┘
luma0to3  (i32x4):  [ R0·6966   | R1·6966   | R2·6966   | R3·6966   ]   products land in i32,
                                                                        nothing can overflow
```

Key point: sum R+G+B in **i32 before rounding**, so there's exactly one rounding error per pixel.

## Step 4: round, shift, narrow back down, store

```ts
luma0to3 = i32x4.shr_s(i32x4.add(luma0to3, q15Half), 15); // +½, then ÷32768 → luma 0..255
// ...same for luma4to7, luma8to11, luma12to15

// pack 16 × i32 back down in two hops, then ONE store writes all 16 pixels
const luma0to7  = i16x8.narrow_i32x4_s(luma0to3,  luma4to7);   // 4+4 × i32 → 8 × i16
const luma8to15 = i16x8.narrow_i32x4_s(luma8to11, luma12to15); // 4+4 × i32 → 8 × i16
v128.store(outPtr, i8x16.narrow_i16x8_u(luma0to7, luma8to15)); // 8+8 × i16 → 16 × u8
```

As TS:

```ts
function i32x4_shr_s(v: Int32Array, bits: number): Int32Array {
  const out = new Int32Array(4); // i32 → Int32Array,  x4 → (4)
  for (let lane = 0; lane < 4; lane++) {
    out[lane] = v[lane] >> bits; // _s → `>>` (sign-keeping shift; _u would be `>>>`)
  }
  return out;
}

// i16x8 = result shape, narrow = squeeze wider lanes down, i32x4 = input shape (×2 of them), _s = signed saturation
function i16x8_narrow_i32x4_s(a: Int32Array, b: Int32Array): Int16Array {
  const out = new Int16Array(8); // i16 → Int16Array,  x8 → (8) ... fed by two i32x4 inputs (4 + 4)
  for (let lane = 0; lane < 8; lane++) {
    const value = lane < 4 ? a[lane] : b[lane - 4];        // first 4 lanes from a, last 4 from b
    out[lane] = Math.min(32767, Math.max(-32768, value));  // saturate: clamp into i16 range, don't wrap
  }
  return out;
}

// same shape one size down: i8x16 = result, i16x8 = inputs (×2), _u = unsigned saturation
function i8x16_narrow_i16x8_u(a: Int16Array, b: Int16Array): Uint8Array {
  const out = new Uint8Array(16); // i8 → Uint8Array,  x16 → (16)
  for (let lane = 0; lane < 16; lane++) {
    const value = lane < 8 ? a[lane] : b[lane - 8]; // first 8 lanes from a, last 8 from b
    out[lane] = Math.min(255, Math.max(0, value));  // saturate into u8 range 0..255
  }
  return out;
}
```

> **Note:**
> Both `narrow` ops **saturate**. A plain TS `Int16Array.from(...)` cast would silently wrap out-of-range values;
> `narrow_..._s` clamps into the i16 range and the final `narrow_..._u` clamps into 0..255. (Luma never exceeds
> 255 here so neither clamp triggers, but it's why these are real ops and not bit-casts.)

## Why this is fast at the silicon level

Every "as TS" function above was a loop over lanes. In silicon, that loop is unrolled in *space* instead of time:
all 16 "iterations" exist as circuits sitting physically side by side, and they all fire in the same cycle.

I assumed the lanes were somehow "connected by gates". It's the opposite: the lanes are deliberately
**disconnected**. The vector ALU is one wide slab of arithmetic hardware, and the lane prefix decides where the
carry chain gets **cut**:

```
one 128-bit add:   [◀━━━━━━━━━━━ carries propagate across all 128 bits ━━━━━━━━━━━]  → 1 result

i8x16.add:         [▮8][▮8][▮8][▮8][▮8][▮8][▮8][▮8][▮8][▮8][▮8][▮8][▮8][▮8][▮8][▮8]  → 16 results
                       ↑ carry chain broken at every 8th bit, same silicon, same cycle

i32x4.add:         [▮━━32━━][▮━━32━━][▮━━32━━][▮━━32━━]                              → 4 results
                       ↑ broken at every 32nd bit instead
```

All 16 byte-adders exist physically side by side and fire in the same cycle. That's half the win. The other half
is amortization: one instruction fetch + decode + dispatch + retire now buys 16 results instead of 1, so the CPU's
front-end (a real bottleneck in tight loops) does 1/16th of the work. And WASM `v128` compiles ~1:1 to native
vector instructions (NEON on Apple Silicon, SSE/AVX on x86), so none of this is lost in translation.

<details markdown="1">
<summary>🔬 Zoom in on the carry wire (optional silicon detour)</summary>

A 128-bit adder is 128 one-bit "full adder" cells in a row. Each cell adds two input bits plus a carry from its
right neighbour, produces one sum bit, and passes a carry to its left neighbour:

```
          a8 b8              a7 b7
           │ │                │ │
          ┌▼─▼─┐   carry     ┌▼─▼─┐   carry
 ... ◀───│ FA8 │◀───────────│ FA7 │◀─────────── ...    sum  = a XOR b XOR c_in
          └─┬──┘             └─┬──┘                     c_out = majority(a, b, c_in)
            ▼                  ▼
           s8                 s7
```

Turning that into a SIMD adder costs almost nothing: put one AND gate on the carry wire at each lane boundary.

```
 ... ◀── FA9 ◀── FA8 ◀──[AND]◀── FA7 ◀── FA6 ◀── ...
                          ▲
                     lane_mode
```

The `lane_mode` signal is decoded from the instruction:

```
i8x16.add  → gate closed at bits 7,15,23,...  carries die at byte edges   → 16 independent adds
i16x8.add  → gate closed at bits 15,31,...    carries die at u16 edges    →  8 independent adds
i32x4.add  → gate closed at bits 31,63,95     carries die at u32 edges    →  4 independent adds
(plain 128-bit add: every gate open, carries ripple all the way, 1 result)
```

So "16 separate adders" and "one wide adder" are the *same physical cells*. A handful of AND gates decides where
one number ends and the next begins. That's all a lane is. (Real adders use carry-lookahead tricks instead of
this textbook ripple chain, and vector units are usually their own datapath — but the lane-cut principle is
exactly the same.)

</details>

Two more pieces of the kernel make sense once you see them as hardware:

- **Why `extmul` comes in low/high halves:** 8 lanes of u16 × u16 produce 8 × 32-bit products. That's 256 bits of
  result and a `v128` register only holds 128, so the op *has* to split into a low half and a high half. It's not
  an API quirk, it's conservation of bits.
- **What a shuffle is:** `i8x16.shuffle` is a byte crossbar. Each of the 16 output byte positions has its own
  multiplexer choosing one of the 32 input bytes, and all 16 muxes select **simultaneously**:

```
 in:   [b0  b1  b2  ... b15 | b16 ... b31]      (two v128 inputs)
          │   │   │       │     │       │
          ▼   ▼   ▼       ▼     ▼       ▼
        ┌──────────── 16 × 32-to-1 byte muxes ───────────┐
        │  mux0   mux1   mux2  ...                mux15  │   each mux's select line
        └───┬──────┬──────┬────────────────────────┬─────┘   comes from the shuffle's
            ▼      ▼      ▼                        ▼          16 immediate indices
 out:  [ b0 ][ b3 ][ b6 ]  ...                  [ b29 ]
```

That's why an arbitrary rearrangement of 16 bytes costs one instruction instead of 16 moves: the bytes aren't
moved "one at a time", every output position grabs its source in the same cycle. The crossbar is also genuinely
expensive silicon (16 × 32-way muxing), which is why shuffles are the op vector ISAs are stingiest about.

## The mental model that finally stuck

Every SIMD kernel is the same four moves: **load → rearrange (shuffle) → widen → multiply/add → narrow → store.**
Most of the "weird" ops (shuffle, extend, narrow) exist only because lanes are fixed-width, they're plumbing to get
data into and out of the width the math needs. The actual arithmetic is the easy part.

Gotchas I hit:
- **`--enable simd`** has to be passed to `asc`, or `v128` doesn't compile.
- **Lane width must match the op.** `i32x4.add` on data you loaded as bytes will silently treat 4 bytes as one int.
- The kernel **silently skips the tail**. The loop condition `i + 16 <= pixels` means the last `pixels % 16`
  pixels are never processed — no crash, no out-of-bounds access, the output bytes just keep whatever stale
  memory was there, which looks like a corrupted last row and throws no error. Real code needs a scalar tail
  loop after the SIMD one; this post gets away without it because the panel widths (800/1872/3840) and the
  bench's 4000×3000 are all multiples of 16.
- Going integer (Q15) made the kernel **bit-deterministic**, the float version drifted ±1 vs the reference across
  platforms, the integer one doesn't.

## Food for thought: isn't this just a tiny GPU?

Kind of, yeah. Somewhere around the third shuffle I realized I was doing by hand the exact job a fragment shader
does for free. A GPU is this same lane trick scaled until it's the whole chip — a "shader core" is a wide SIMD
unit (NVIDIA's warps: 32 lanes marching in lock-step), and the whole point of the shader programming model is
that you write code A and the hardware runs code B. All the plumbing in this post — de-interleave, widen,
narrow — is what a GPU quietly does so you never have to know lanes exist.

So the ladder, as I think of it now:

```
scalar loop    1 px per instruction      code A
SIMD           16 px per instruction     code B   ← you are here
workers        × cores, same kernel
GPU            thousands of px in flight... in another building
```

"Another building" is the catch. The pixels have to commute: into GPU memory, through a dispatch queue, and back
out. Those costs are fixed and don't care that the math is three multiplies per pixel. Does the commute eat the
win for a one-pass 5.7 ms kernel? Honestly: no idea, I haven't benchmarked it (WebGPU exists in Deno — future
TIL, maybe). My hunch is that data already in hand + tiny math per byte favors SIMD, and heavy math or data that
stays resident across passes favors the GPU. One thing I do know: the bit-exactness I got from integer lanes is
not something float shaders promise across vendors.
