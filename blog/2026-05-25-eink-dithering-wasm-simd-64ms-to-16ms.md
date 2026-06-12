# Dithering an e-ink frame in 16ms instead of 64ms with WASM + SIMD

My TRMNL e-ink display runs off [my own Deno BYOS server](https://github.com/hertzg/trmnl_byos_deno). The render
pipeline is: lay the dashboard out as HTML, screenshot it in headless Chrome over CDP, then turn that RGB screenshot
into the 1/2/4-bit grayscale the panel actually wants. That last step (grayscale + Floyd-Steinberg dithering) was
the slowest thing in the whole request, and it was pure JS.

After this rabbit hole the dither pipeline does an 1800×1480 4bpp frame in ~15.7ms instead of ~64ms (M2 Pro), the
working set fits in L1, and the kernel is bit-exact against the JS reference it replaced.

Notes from the journey, mostly so future me remembers why this kernel looks the way it does.

## What "dithering" is doing here

E-ink at 1bpp can only show black or white. To fake a gray it scatters black and white pixels so the *average*
looks gray. Floyd-Steinberg is the classic: walk the pixels left-to-right, top-to-bottom, snap each one to the
nearest available level, and push the rounding error onto the not-yet-visited neighbours with these weights:

```
        *   7/16
  3/16 5/16 1/16
```

Two things matter for performance. The luma step (RGB → one gray value per pixel) is **embarrassingly parallel**,
every pixel is independent, perfect for SIMD. The Floyd-Steinberg step is the opposite: pixel `x` can't start until
pixel `x-1` has pushed its error, so the row is a **serial dependency chain**. The whole optimisation is about
making the parallel part fly and the serial part cheap.

## The fixes, in order of discovery

### 1. Fuse the whole thing into one WASM kernel (64ms → 24ms, 2.83× vs JS)

The JS version ran as stages: decode to RGBA, build an f32 luma buffer, run Floyd-Steinberg over it, pack. Each
stage streamed the whole image through memory again.

I collapsed luma + Floyd-Steinberg into a single AssemblyScript kernel that takes RGB in and writes quantized
indices out, no intermediate luma buffer. Then I leaned on it:

- f32 throughout (the JS used f64 it didn't need)
- every per-pixel divide constant-folded into a reciprocal multiply
- SIMD luma with `f32x4` and `i8x16` channel shuffles (`--enable simd` in the asc flags)
- the three contiguous down-row error writes packed into one `v128` store

Funny detail: the *first* working WASM version was **slower than the JS** (69ms vs 64ms). WASM isn't magic; a naive
port just moves the same memory traffic behind a FFI boundary. The wins were all in what came after.

### 2. Two-row staging + an accumulator trick (24ms → 17ms, 3.59× vs JS)

The big one. The old kernel kept a `(W+2)·(H+1)` f32 error arena (about **1.5 MB** for TRMNL X) and streamed it
through L2 twice per frame. I replaced it with **two `(W+2)` i16 row buffers**, ~7 KB total, small enough to live in
L1. Row `y+1`'s luma gets computed straight into the "next" buffer one row ahead of the Floyd-Steinberg reader;
after each row the two buffers just swap pointers.

Then the part I'm smug about (though the idea isn't mine, it's straight out of Itamar Turner-Trauring's
[*Speeding up your code when multiple cores aren't an option*](https://pythonspeed.com/articles/optimizing-dithering/),
which argues Floyd-Steinberg is basically impossible to parallelize and the real win is keeping the error in
registers instead of hammering a memory array. Each cell in the row below collects error from **three** pixels above
it (its down-left, down, and down-right neighbours). The obvious code writes three f32 cells per pixel. Instead I
carry the un-divided error sum in two scalar registers across the loop:

```
next[x]  =  1·err(x-2)  +  5·err(x-1)  +  3·err(x)
              └────────────┬──────────┘     └─ this pixel
            carried in dlPrev, un-divided    ×3, added at commit
```

So each pixel commits **one** i16 cell and does **one** divide (a `>>4` shift, because the weights sum to 16),
instead of three writes and three divides. 16× less down-row write bandwidth, and one rounding point per cell
instead of three.

Going all-integer had a bonus I didn't expect: the kernel became **bit-deterministic**. Round-half-up via
`(x + bias) / step` matches JS `Math.round` exactly, so the parity test went from "visually identical (≤1 level on
≤1% of pixels)" back to genuinely bit-exact, including the all-black and all-white cases the f32 path could only
get "close" on.

### 3. Stop carrying an alpha channel nobody wanted (dither block 109ms → 61ms warm)

The screenshot comes from CDP's `Page.captureScreenshot`, and CDP **always** emits the same PNG: 8-bit colour
type 2 (RGB), non-interlaced, level-1 zlib. A general PNG decoder handles a grab-bag of formats it'll never see
here. So I wrote a decode specialized on exactly those invariants: inflate via `DecompressionStream`, defilter
straight into RGB output, no ping-pong scratch, no RGB→RGBA widen. 2.9× faster than the general decoder, and the
kernel lost a quarter of its input bytes (and all the wasted alpha math) by going `ditherFromRgba` → `ditherFromRgb`.

### 4. Planar 16-px luma in Q15 fixed-point (17ms → 15.7ms, and bit-exact)

The luma SIMD was reading 8 pixels per chunk with overlapping loads. I rewrote it to do **16 pixels per chunk**:
three back-to-back `v128` loads (48 bytes, every byte used, no overlap), a two-stage `i8x16.shuffle` that
deinterleaves the `RGBRGB…` stream into separate `R`, `G`, `B` byte planes, then luma computed in **Q15
fixed-point**.

Q15 just means: instead of multiplying by `0.2126` in float, multiply by `round(0.2126 × 2^15)` as an integer and
shift back down by 15 at the end. The Rec.709 weights become `6966 / 23436 / 2366`, they sum to exactly 32768, and
`i32x4.extmul_*_i16x8_u` gives the widening multiply for free. R, G and B sum in i32 *before* a single
`(+0x4000) >> 15` round.

That dropped the luma pass from ~3.75 to ~2.94 SIMD ops/pixel, and because it's now integer, made it bit-exact
with the float reference across every bit depth and panel size, including odd widths that exercise the scalar tail.
The f32 path had always needed ±1 ULP of slack; this one doesn't.

### 5. The cheap wins

A handful of small ones that needed no cleverness:

- **Memoize the memory layout.** Production renders the same frame size every time, so the layout calc collapses to
  two integer compares and a cached pointer struct. WASM memory grows at most once, ever.
- **`memory.fill` to zero the last row's staging** instead of a scalar loop.
- **One CDP `Page`, reused.** Same memoization I already had for the browser: open the page once, `goto` for each
  render instead of new-page / close every time. Drops two CDP round-trips off every frame.
- **Delete the JS path from production.** It had been dead weight behind a toggle nobody flipped. It stays in the
  tree though, it's the parity oracle the WASM kernel is tested against, and the baseline in the benchmark.

### 6. A coda: debugging WASM you can actually step through

Once the kernel was the hot path I wanted to step it in DevTools. AssemblyScript emits a source map, but Deno's
`raw-imports` instantiates the module from bytes, so there's no URL for Chrome to resolve `./dither.wasm.map`
against (it tries `wasm://wasm/dither.wasm.map` and fails). Two fixes made it work: build with `--baseDir` so the
map records source paths relative to itself, and embed an **absolute `file://`** `sourceMappingURL` so DevTools
fetches the map directly. Now breakpoints land in the `.as.ts`.

## Measuring honestly

- **The first WASM port was slower than JS.** If I'd measured only "did WASM help" and stopped, I'd have shipped a
  regression. The wins were entirely in the iterations after.
- I kept the **native JS pipeline as a parity oracle** the whole way. Every speed change had to stay bit-exact (or
  within the documented visual tolerance) against it, checked across 1/2/4 bpp, both panel shapes, all-black,
  all-white, isolated channels, and odd widths. Performance work that quietly changes output isn't a win, it's a bug.
- Per-phase benchmark splits copy-in, kernel-only, and the full round-trip, so I could see *which* part moved
  instead of staring at one end-to-end number.

## Where it landed

1800×1480 @ 4bpp, M2 Pro:

| | before | after |
|---|---|---|
| dither pipeline (luma + Floyd-Steinberg) | ~64ms (JS) | **~15.7ms** |
| kernel only | ~60.5ms | **~15.4ms** |
| Floyd-Steinberg working set | ~1.5 MB f32 arena | **~7 KB** (two i16 rows, L1-resident) |
| PNG decode | general decoder | **2.9× faster** (CDP-specialized) |
| luma SIMD throughput | ~3.75 ops/px | **~2.94 ops/px** |
| output vs JS reference | "visually identical" | **bit-exact** |

## What's left and why I stopped

The luma pass is already SIMD and near its op floor. The Floyd-Steinberg pass is **inherently serial**, each pixel
depends on its left neighbour's error, so you can't vectorize across the row, and the scalar body is sitting at the
inter-pixel dependency-chain floor. Going faster from here means a *different* algorithm (a parallel-friendly dither,
or quantizing on the GPU), not a faster Floyd-Steinberg. For a frame that refreshes every few minutes on an e-ink
panel, 16ms is so far under the budget that the next bottleneck is Chrome taking the screenshot. So I stopped.

## References

- Itamar Turner-Trauring, [*Speeding up your code when multiple cores aren't an option*](https://pythonspeed.com/articles/optimizing-dithering/).
  The register-accumulator idea behind the kernel; also the parity reference I matched bit-for-bit (down to dropping
  the last pixel's down-right error at the right border).
- R.W. Floyd & L. Steinberg, *An Adaptive Algorithm for Spatial Greyscale* (1976), the original error-diffusion paper.
- ITU-R BT.709, the `0.2126 / 0.7152 / 0.0722` luma weights (my `6966 / 23436 / 2366` Q15 constants).
- [WebAssembly fixed-width SIMD](https://github.com/WebAssembly/simd) and the
  [AssemblyScript SIMD docs](https://www.assemblyscript.org/stdlib/globals.html#sim---128-bit-vectors), every
  `v128` op used (`i8x16.shuffle`, `extend_*`, `extmul_*`, `narrow`, `splat`).
- [Chrome DevTools Protocol `Page.captureScreenshot`](https://chromedevtools.github.io/devtools-protocol/tot/Page/#method-captureScreenshot),
  the 8-bit RGB / level-1 zlib invariant the specialized PNG decoder exploits.
- [Source Map spec](https://tc39.es/source-map/) and
  [Chrome DevTools WASM debugging](https://developer.chrome.com/docs/devtools/wasm), for the
  step-through-the-`.as.ts` coda.
