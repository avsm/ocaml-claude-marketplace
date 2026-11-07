---
name: oxcaml-simd
description: "OxCaml SIMD vector types and intrinsics for SSE/AVX (and NEON) high-performance parallel numeric operations"
---

# OxCaml SIMD: Detailed Guide

OxCaml has built-in SIMD vector types and intrinsics. The typed
intrinsics library (`ocaml_simd`) is modelled on x86-64
(SSE/AVX/AVX2). It also builds and runs on arm64, where 128-bit
vectors are native (NEON) and the operations without a NEON builtin go
through C stubs — see "arm64 / NEON Status". 512-bit (AVX-512) types
exist behind `-extension simd_beta` but have no intrinsics yet.

## Vector Types

| Element type | 128-bit (SSE / NEON) | 256-bit (AVX) |
|--------------|----------------------|---------------|
| 8-bit int | `int8x16` (16 elements) | `int8x32` (32) |
| 16-bit int | `int16x8` (8) | `int16x16` (16) |
| 32-bit int | `int32x4` (4) | `int32x8` (8) |
| 64-bit int | `int64x2` (2) | `int64x4` (4) |
| 32-bit float | `float32x4` (4) | `float32x8` (8) |
| 64-bit float | `float64x2` (2) | `float64x4` (4) |

Each has an unboxed version with a `#` suffix (`int8x16#`,
`float64x4#`, …).

The half-float vectors `float16x8` / `float16x16` are also available
by default. The **512-bit** family (`int8x64` … `float64x8`,
`float16x32`, kind `vec512`) and the `mask` / `mask#` types need
`-extension simd_beta` — see "AVX-512" below.

The vector type names (`int32x4#` etc.) were **not** affected by the
removal of `int64#` / `int32#` / `nativeint#` / `float32#`. Scalar
lane types in signatures are spelled `int64_u`, `int32_u`,
`nativeint_u`, `float32_u` (`float#`, `int8#`, `int16#` unchanged).

### Boxed vs Unboxed

- **Unboxed** (`#` suffix): passed in XMM/YMM registers, stored flat in
  records (`type vec4 = { data : float32x4# }`), can be in flat
  arrays, no heap allocation for the vector itself.
- **Boxed** (no suffix): heap-allocated when passed to non-inlined
  functions (storing one, `global_ref := v`, allocates), may generate
  unaligned load/store, easier to use in polymorphic code. Boxed
  vectors use tag 0 in the block header (amd64 and arm64): code that
  inspects vector block tags via `Obj.tag` or by reading the header
  directly will observe that value.
- The library modules work on the **unboxed** types
  (`Float32x4.t = float32x4#`); convert with `Float32x4.box` / `unbox`.
- Arrays of unboxed pairs of `vec128` are allowed, laid out as flat
  pairs: `#(int32x4# * int32x4#) array`.

## Library Setup

The opam package is `ocaml_simd` (at `v0.18~preview.133.03+642` it
declares `conflicts: "oxcaml-compiler" {< "5.4.0-ox9"}`). The
libraries you use, referenced by public name in dune:

```lisp
(libraries ocaml_simd.sse)            ; module Ocaml_simd_sse (128-bit)
; Or for AVX/AVX2 (re-exports the 128-bit modules too):
(libraries ocaml_simd.avx)            ; module Ocaml_simd_avx
; For [%shuffle ...] / [%blend ...] / [%permute ...] constants:
(preprocess (pps ocaml_simd.ppx))
```

There are no separate `ocaml_simd_sse` / `ocaml_simd_avx` / `ppx_simd`
opam packages, and nothing to `open` for the ppx. Both libraries are
`(modes native)`.

**`open` shadows Stdlib modules.** `Ocaml_simd_sse` exports its own
`Int16`, `Int32`, `Int64` and `String` (scalar intrinsics such as
`Int64.count_set_bits`, and the SSE4.2 string comparisons); `Ocaml_simd_avx`
exports `Float32`, `Float64`, `Int16`, `Int32`, `Int64`. After
`open Ocaml_simd_sse`, `String.make` is `Unbound value String.make` —
write `Stdlib.String.…` / `Stdlib.Int64.…`, or don't open the library
and alias the vector modules you need
(`module Float32x4 = Ocaml_simd_sse.Float32x4`).

**arm64: the opam package does not build as released.**
`opam install ocaml_simd` (`v0.18~preview.133.03+642`) fails on arm64
with `sse_stubs.c: fatal error: 'simde/x86/sse.h' file not found`: the
arm64 C stubs include the vendored `simde` headers, which the build
does not put on the include path. Pointing the C compiler at the
headers in opam's own build directory gets it installed (checked on
macOS arm64):

```sh
CPATH="$(opam var prefix)/.opam-switch/build/ocaml_simd.v0.18~preview.133.03+642/simde" \
  opam install ocaml_simd
```

x86-64 does not take that code path.

## Basic Operations

### Creating and Extracting

```ocaml
open Ocaml_simd_sse

(* Arguments are UNBOXED scalars (float32_u here), lane 0 first *)
let v = Float32x4.set #1.0s #2.0s #3.0s #4.0s
(* lanes: [1.0; 2.0; 3.0; 4.0] *)

(* Same value in all lanes *)
let v = Float32x4.set1 #5.0s
(* lanes: [5.0; 5.0; 5.0; 5.0] *)

(* Static vector literals (arguments must be literals) *)
let k = Float32x4.const #1.0s #2.0s #3.0s #4.0s
let ones = Float32x4.const1 #1.0s          (* also Float32x4.zero / one *)

(* All lanes, as an unboxed tuple (slow; for debugging/printing) *)
let #(a, b, c, d) = Float32x4.splat v

(* Single lane: labelled index is an int64_u in [0,3] *)
let first = Float32x4.extract ~idx:#0L v
let last = Float32x4.extract ~idx:#3L v

(* Lane 0 is free *)
let first = Float32x4.extract0 v            (* : float32_u *)
```

`set` takes one unboxed scalar per lane — `float32_u` for `Float32x4`,
`float#` for `Float64x2`, `int32_u` for `Int32x4`, etc. Passing boxed
literals (`Float32x4.set 1.0 2.0 3.0 4.0`) is a type error. There is
no `broadcast` (use `set1`) and no `of_tuple`.

### Arithmetic, Min/Max, Comparisons

```ocaml
let a = Float32x4.set #1.0s #2.0s #3.0s #4.0s
let b = Float32x4.set #5.0s #6.0s #7.0s #8.0s

let sum = Float32x4.add a b       (* [6, 8, 10, 12] *)
let diff = Float32x4.sub a b      (* [-4, -4, -4, -4] *)
let prod = Float32x4.mul a b      (* [5, 12, 21, 32] *)
let quot = Float32x4.div a b      (* [0.2, 0.33, 0.43, 0.5] *)
let sum' = Float32x4.(a + b)      (* operators + - * / also exist *)

(* Math functions *)
let roots = Float32x4.sqrt a      (* [1, 1.41, 1.73, 2] *)
let recip = Float32x4.rcp a       (* Approximate reciprocal *)
let rsqrt = Float32x4.rsqrt a     (* Approximate 1/sqrt *)

let mins = Float32x4.min a b      (* Element-wise min *)
let maxs = Float32x4.max a b      (* Element-wise max *)

(* Comparisons are operators; the result is a mask (int32x4# here) *)
let mask = Float32x4.(a < b)      (* a < b per element *)
let mask = Float32x4.equal a b    (* or Float32x4.(a = b) *)
let mask = Float32x4.(a > b)      (* also >=, <=, <>, is_nan *)

(* Use a mask *)
let picked = Float32x4.select mask ~fail:a ~pass:b
let bits = Float32x4.movemask mask          (* : int64_u *)
```

### Integer Operations

```ocaml
open Ocaml_simd_sse

let a = Int32x4.set #1l #2l #3l #4l          (* int32_u lanes *)
let b = Int32x4.set #5l #6l #7l #8l

(* Arithmetic *)
let sum = Int32x4.add a b
let diff = Int32x4.sub a b

(* Bitwise: operators, not and_/or_/xor *)
let anded = Int32x4.(a land b)
let ored = Int32x4.(a lor b)
let xored = Int32x4.(a lxor b)
let inv = Int32x4.lnot a

(* Shifts by a runtime count (int64_u) *)
let left = Int32x4.shift_left_logical a #2L
let right = Int32x4.shift_right_logical a #1L
let arith = Int32x4.shift_right_arithmetic a #1L
(* Immediate-count forms: Int32x4.shifti_left_logical #2L a, etc. *)
```

### Horizontal Operations

Two `horizontal_add`s (pairwise adds) leave the complete sum in lane 0:

```ocaml
let dot4 (a : float32x4#) (b : float32x4#) : float32_u =
  let prod = Float32x4.mul a b in
  let sum1 = Float32x4.horizontal_add prod prod in
  let sum2 = Float32x4.horizontal_add sum1 sum1 in
  Float32x4.extract0 sum2

(* or simply (SSE4.1 dpps) *)
let dot4' a b : float32_u = Float32x4.dot a b
```

### AVX Operations

```ocaml
open Ocaml_simd_avx

(* 256-bit vectors - double the width; float# lanes *)
let v = Float64x4.set #1.0 #2.0 #3.0 #4.0
let w = Float64x4.set #5.0 #6.0 #7.0 #8.0

let sum = Float64x4.add v w  (* 4 doubles in parallel *)

(* 256-bit integer vectors: eight int32_u *)
let i = Int32x8.set #1l #2l #3l #4l #5l #6l #7l #8l

(* Build from / split into 128-bit lanes *)
let j = Int32x8.set_lanes (Int32x4.set1 #1l) (Int32x4.set1 #2l)
let lo = Int32x8.extract_lane0 j
```

## Shuffles and Blends

Shuffle/blend/permute selectors must be compile-time constants; the
`ocaml_simd.ppx` rewriter builds them (`[%shuffle …]`, `[%blend …]`,
`[%permute …]`). Shuffle and blend are compiler builtins on **amd64
only**; on arm64 the same calls work but go through the library's C
stubs.

```ocaml
let v = Float32x4.set #1.0s #2.0s #3.0s #4.0s

(* _mm_shuffle_ps: four indices in [0,3]; the low two result lanes
   are picked from the first vector, the high two from the second *)
let shuffled = Float32x4.shuffle [%shuffle 3, 2, 1, 0] v v

let broadcast_first = Float32x4.shuffle [%shuffle 0, 0, 0, 0] v v
(* Result: all four lanes = lane 0 of v *)
```

```ocaml
(* _mm_blend_ps: per lane, 0 = from the first vector, 1 = from the second *)
let a = Float32x4.set #1.0s #2.0s #3.0s #4.0s
let b = Float32x4.set #5.0s #6.0s #7.0s #8.0s

let blended = Float32x4.blend [%blend 0, 1, 0, 1] a b
```

For a runtime (non-constant) choice use `Float32x4.select mask
~fail ~pass`.

## Load and Store

### From Arrays

Use `floatarray` or `float# array`:

```ocaml
let arr : float# array = [| #1.0; #2.0; #3.0; #4.0; #5.0; #6.0; #7.0; #8.0 |]

(* Load 2 unboxed floats starting at index 0 *)
let v1 = Float64x2.Float_u_array.get arr ~idx:0
let v2 = Float64x2.Float_u_array.get arr ~idx:2

(* Store back *)
Float64x2.Float_u_array.set arr ~idx:0 result

(* floatarray *)
let fa = Float.Array.make 8 0.0
let v3 = Float64x2.Floatarray.get fa ~idx:0
```

Each vector module exposes the accessors that make sense for it:
`Float64x2.Floatarray` / `Float_u_array`, `Float32x4.Float32_u_array`,
`Int32x4.Int32_u_array`, `Int8x16.Int8_u_array`, … (with `unsafe_get`
/ `unsafe_set` variants, and `Int8_u` … `Nativeint_u` submodules for
other index types).

**SIMD access to plain `float array` / `float iarray` does not exist**
(it cannot work under `--disable-flat-float-array`). There is no
`Float_array` accessor module, and the compiler primitives
`%caml_float_array_get128/set128` (and the 256/512-bit, `u`, `#` and
`_indexed_by_*` variants) are gone:

```
external get : float array -> int -> float64x2 = "%caml_float_array_get128"
Error: Unknown builtin primitive %caml_float_array_get128
```

The surviving array families, each for 128/256/512 bits, safe/`u`,
boxed/`#` result and six index kinds:

| Array type | Primitive stem |
|------------|----------------|
| `floatarray` | `%caml_floatarray_get128` |
| `float# array` | `%caml_unboxed_float_array_get128` |
| `float32_u array` | `%caml_unboxed_float32_array_get128` |
| `int array` | `%caml_int_array_get128` |
| `int64_u array` | `%caml_unboxed_int64_array_get128` |
| `int32_u array` | `%caml_unboxed_int32_array_get128` |
| `int16# array` | `%caml_untagged_int16_array_get128` |
| `int8# array` | `%caml_untagged_int8_array_get128` |
| `nativeint_u array` | `%caml_unboxed_nativeint_array_get128` |

**Primitive-name strings keep their `#`** even though the *types* were
renamed: the index-typed variants are still spelled
`_indexed_by_int32#` etc., and `#` still marks an unboxed result.

```ocaml
external bget_i32 : bytes -> int32_u -> int8x16
  = "%caml_bytes_getu128_indexed_by_int32#"      (* int32_u in the type *)
external bget_u : bytes -> int -> int8x16# = "%caml_bytes_getu128#"
```

### From Strings/Bytes

```ocaml
let bytes = Bytes.create 64

(* Load 16 bytes as int8x16 (bounds-checked; unsafe_get skips it) *)
let v = Int8x16.Bytes.get bytes ~byte:0

(* Store 16 bytes *)
Int8x16.Bytes.set bytes ~byte:0 v

(* Strings are read-only *)
let w = Int8x16.String.get text ~byte:0
```

### From Bigstring

The bigstring accessors take a plain
`(char, int8_unsigned_elt, c_layout) Bigarray.Array1.t` and make the
alignment explicit:

```ocaml
let buf = Bigarray.(Array1.create char c_layout 128)

let v = Float64x2.Bigstring.unaligned_get buf ~byte:0
let w = Float64x2.Bigstring.aligned_get buf ~byte:16   (* raises if misaligned *)
Float64x2.Bigstring.unaligned_set buf ~byte:32 result
Float64x2.Bigstring.aligned_set buf ~byte:48 result
(* unsafe_unaligned_get / unsafe_aligned_get / ..._set skip the checks *)
```

A malloc-backed bigstring is 16-byte aligned, so aligned access at
16-byte offsets is valid.

### Process Array in Chunks

The stdlib `Array.length` / `get` / `set` only accept `value`
elements, so on a `float# array` use Base (`open Base`, whose `Array`
accessors are layout-polymorphic) or declare layout-poly externals:

```ocaml
external len : ('a : any mod separable). 'a array @ local -> int
  = "%array_length" [@@layout_poly]
external get : ('a : any mod separable). 'a array -> int -> 'a
  = "%array_safe_get" [@@layout_poly]
external set : ('a : any mod separable). 'a array -> int -> 'a -> unit
  = "%array_safe_set" [@@layout_poly]

let process_array (arr : float# array) : unit =
  let n = len arr in
  let vec_len = n / 2 * 2 in

  (* Process 2 doubles at a time *)
  for i = 0 to vec_len / 2 - 1 do
    let v = Float64x2.Float_u_array.get arr ~idx:(i * 2) in
    let result = Float64x2.sqrt v in
    Float64x2.Float_u_array.set arr ~idx:(i * 2) result
  done;

  (* Handle remainder *)
  for i = vec_len to n - 1 do
    set arr i (Stdlib_upstream_compatible.Float_u.sqrt (get arr i))
  done
```

## Raw Compiler Builtins

Normally you call builtins through the `ocaml_simd` libraries. A
builtin is an `external` with `[@@builtin]`, a bytecode stub name
(may be `""`) and the builtin name (`caml_`-prefixed). The builtin
name is still referenced as a native symbol, so it must resolve at
link time (`ocaml_simd` and the compiler tests define a dummy C
function per builtin).

### Load/Store Intrinsics

Raw loads/stores take a **raw address** (`nativeint_u`), not `bytes`
(the library wraps these as `Float32x4.Raw.aligned_load` etc.):

```ocaml
type addr = nativeint_u

(* Aligned load/store - requires 16/32-byte alignment *)
external load_aligned : addr -> (int64x2[@unboxed])
  = "" "caml_sse_vec128_load_aligned" [@@noalloc] [@@builtin]
external store_aligned : addr -> (int64x2[@unboxed]) -> unit
  = "" "caml_sse_vec128_store_aligned" [@@noalloc] [@@builtin]

(* Unaligned load/store - works with any alignment *)
external load_unaligned : addr -> (int64x2[@unboxed])
  = "" "caml_sse_vec128_load_unaligned" [@@noalloc] [@@builtin]
```

Also: `caml_sse_vec128_store_unaligned`,
`caml_sse_vec128_store_aligned_uncached` (non-temporal),
`caml_sse41_vec128_load_aligned_uncached`,
`caml_sse2_vec128_load_low64` / `_load_low32` (partial loads),
`caml_sse2_vec128_store_low64` / `_low32` / `_mask8`, and the
`caml_avx_vec256_load_*` / `_store_*` family.

### AVX2 Gather Intrinsics

Gathers load multiple values from non-contiguous memory addresses.
They are named by lane width × index width — there are no float-typed
gathers (reinterpret lanes as needed):

```ocaml
(* vec{128,256}_gather{32,64}_index{32,64} *)
external vec128_gather32_index32
  : scale:int64_u -> onto:(int32x4[@unboxed]) -> addr
  -> idx:(int32x4[@unboxed]) -> mask:(int32x4[@unboxed])
  -> (int32x4[@unboxed])
  = "" "caml_avx2_vec128_gather32_index32" [@@noalloc] [@@builtin]

(* Scale must be 1, 2, 4, or 8 (an int64_u literal, e.g. ~scale:#4L) *)
```

### BMI/BMI2 Intrinsics

Bit manipulation instructions, available for `int32` and `int64`
(`[@@unboxed]`, or directly on `int32_u` / `int64_u`):

```ocaml
(* ANDN: Bitwise AND of inverted first operand with second *)
external andn : int64 -> int64 -> int64
  = "caml_vec128_unreachable" "caml_bmi_andn_int64"
  [@@noalloc] [@@unboxed] [@@builtin]

(* Same builtin on the unboxed type *)
external andn_u : int64_u -> int64_u -> int64_u
  = "caml_vec128_unreachable" "caml_bmi_andn_int64"
  [@@noalloc] [@@builtin]

(* RORX: rotate count first, untagged *)
external rorx : (int[@untagged]) -> (int64[@unboxed]) -> (int64[@unboxed])
  = "caml_vec128_unreachable" "caml_bmi2_rorx_int64"
  [@@noalloc] [@@builtin]
```

Builtin names (each with `_int32` and `_int64`; the int16 builtins are
`caml_popcnt_int16`, `caml_lzcnt_int16` and `caml_bmi_tzcnt_int16`):

| Set | Builtins |
|-----|----------|
| BMI | `caml_bmi_andn`, `caml_bmi_bextr` (bit field extract), `caml_bmi_blsi` (lowest set bit), `caml_bmi_blsmsk` (mask up to lowest set bit), `caml_bmi_blsr` (reset lowest set bit), `caml_bmi_tzcnt` (also `_int16`) |
| BMI2 | `caml_bmi2_bzhi` (zero high bits), `caml_bmi2_mulx`, `caml_bmi2_pdep`, `caml_bmi2_pext`, `caml_bmi2_rorx`, `caml_bmi2_sarx` / `_shrx` / `_shlx` |
| POPCNT / LZCNT | `caml_popcnt`, `caml_lzcnt` (also `_int16`) |

### 128-bit Integer Arithmetic

```ocaml
(* (low, high) + (low, high) -> (low, high) *)
external add_int128
  : x0:int64_u -> x1:int64_u -> y0:int64_u -> y1:int64_u
  -> #(int64_u * int64_u)
  = "" "caml_int128_add" [@@noalloc] [@@builtin]
(* likewise "caml_int128_sub" *)
```

## AVX-512 (`-extension simd_beta`)

AVX-512 support is **types and plumbing only**: you can declare,
store, load and pass 512-bit vectors and masks, but there are
**no AVX-512 intrinsics** (no `caml_avx512_*` builtin) and no 512-bit
modules in `ocaml_simd`. Real work on these values must go through C
stubs.

| Item | How to get it |
|------|---------------|
| `int8x64 int16x32 int32x16 int64x8 float16x32 float32x16 float64x8` (+ `#` versions) | `-extension simd_beta` (was `simd_alpha` on minus-39) |
| `mask` (boxed), `mask#` (unboxed; k0–k7, 64 bits) | `-extension simd_beta` |
| kinds `vec512`, `mask` | always available |

Without the flag: `Error: Unbound type constructor int8x64`.
`-extension-universe beta` also enables them; `stable` does not.

```ocaml
(* ocamlopt -extension simd_beta *)
external box_mask   : mask# -> mask = "%box_mask"
external unbox_mask : mask -> mask# = "%unbox_mask"

external mask_of_int64 : int64 -> mask
  = "caml_vec512_unreachable" "caml_mask_of_int64"
  [@@noalloc] [@@unboxed] [@@builtin]
external int64_of_mask : mask -> int64
  = "caml_vec512_unreachable" "caml_int64_of_mask"
  [@@noalloc] [@@unboxed] [@@builtin]

(* load/store *)
external get_mask : bytes -> int -> mask = "%caml_bytes_getmask"
external get512 : bytes -> int -> int64x8 = "%caml_bytes_getu512"

(* records, arrays *)
type r = { v : int64x8#; m : mask#; s : string }
external v_get : int64x8# array -> int -> int64x8# = "%array_safe_get"
```

- Other builtins: `caml_vec512_cast`,
  `caml_vec512_low_{of,to}_vec128`, `caml_vec512_low_{of,to}_vec256`;
  `%box_vec512` / `%unbox_vec512`.
- Mask load/store primitives: `%caml_bytes_getmask` / `_setmask`,
  `%caml_string_getmask`, `%caml_bigstring_getmask` / `_setmask`
  (each with optional `u`, `#` and `_indexed_by_{int8#,…,nativeint#}`
  variants). There is no string setter.
- `int64x8# array` and `mask# array` work through the layout-poly
  `%array_safe_get` / `%array_safe_set` / `%array_length` primitives.
- **Backend flags (amd64 only)**: `-favx512f`, `-favx512dq`,
  `-favx512cd`, `-favx512bw`, `-favx512vl` (and `-fno-…`). They imply
  each other and AVX2, and are **off by default** regardless of the
  build host. On arm64 the flags do not exist
  (`ocamlopt: unknown option '-favx512f'`).
- **C stubs**: `(int64x8[@unboxed])`, `(mask[@unboxed])` and the bare
  `int64x8#` / `mask#` types are accepted in externals. Masks cross
  the C ABI in general-purpose registers (`__mmask8` … `__mmask64`).
  `caml/simd.h` provides, under `ARCH_AVX512` (`__AVX512F__`):
  `Vec512_val` / `Vec512_vald` / `Vec512_vali`, `Store_vec512_val*`,
  `caml_copy_vec512` / `512d` / `512i` (compile the C with
  `-mavx512f …`). There is no mask helper.

## C FFI

```ocaml
(* Unboxed external - no allocation overhead *)
external simd_process : (float32x4[@unboxed]) -> (float32x4[@unboxed]) =
  "boxed_simd_process" "unboxed_simd_process"

(* Equivalent, using the unboxed type directly *)
external simd_process' : float32x4# -> float32x4# =
  "boxed_simd_process" "unboxed_simd_process"

(* For 256-bit *)
external avx_process : (float64x4[@unboxed]) -> (float64x4[@unboxed]) =
  "boxed_avx_process" "unboxed_avx_process"
```

```c
#include <caml/simd.h>
#include <immintrin.h>

// Unboxed version - efficient
__m128 unboxed_simd_process(__m128 v) {
  return _mm_sqrt_ps(v);
}

// Boxed wrapper
CAMLprim value boxed_simd_process(value v) {
  return caml_copy_vec128(unboxed_simd_process(Vec128_val(v)));
}

// For 256-bit
__m256d unboxed_avx_process(__m256d v) {
  return _mm256_sqrt_pd(v);
}

CAMLprim value boxed_avx_process(value v) {
  return caml_copy_vec256d(unboxed_avx_process(Vec256_vald(v)));
}
```

On arm64 the same pattern works with `<arm_neon.h>` types
(`float32x4_t`, `float64x2_t`, …); `caml/simd.h` provides
`Vec128_val*` / `caml_copy_vec128*` under `__ARM_NEON`.

## arm64 / NEON Status

- **128-bit vectors work natively**: the types, `%box_vec128` /
  `%unbox_vec128`, the load/store `%` primitives, records, arrays and
  C stubs all compile and run on arm64.
- arm64 has **its own builtins** (about 250, mostly `caml_neon_*`,
  some `caml_simd_*`):

  ```ocaml
  external add : float32x4 -> float32x4 -> float32x4
    = "caml_vec128_unreachable" "caml_neon_float32x4_add"
    [@@noalloc] [@@unboxed] [@@builtin]
  ```

- An **x86 builtin used on arm64** is a compile error at the call
  site (the declaration alone is accepted):

  ```
  Error: External annotated with [@@builtin] is not recognized: "caml_sse_float32x4_add"
  ```

- **`ocaml_simd` works on arm64, both `ocaml_simd.sse` and
  `ocaml_simd.avx`** (once installed — see "Library Setup" for the
  opam build failure). Roughly 120 of its externals name an arm64
  builtin (`[@@builtin (amd64, "caml_sse_float32x4_add") (arm64,
  "caml_neon_float32x4_add")]`); nearly 600 are
  `[@@builtin amd64]`, which on arm64 means an ordinary `[@@noalloc]`
  C call into stubs implemented with the vendored `simde`. That
  covers `shuffle`, `blend`, gathers and every
  256-bit operation: results are correct, but each such operation is
  a function call rather than one instruction.
- **256-bit vectors are lowered to tuples** during codegen (bridged by
  `Preinterpret_boxed_vector_as_tuple` /
  `Preinterpret_tuple_as_boxed_vector`): they compile and run, but at
  a representation cost and with no native 256-bit instructions —
  prefer 128-bit operations where arm64 performance matters.
- **Unboxed 512-bit vectors and `mask#` crash the arm64 backend** for
  any function that takes, returns or stores one (`Fatal error:
  arm64: got 256/512 bit vector or mask`), and so does calling a
  512-bit or mask load/store `%` primitive, even with a boxed result.
  Type declarations, `external` declarations and passing boxed values
  around are fine.

## CPU Features, Limitations, Performance

AVX/AVX2 availability is a **compile-time** property of the compiler
build (`Config.has_avx` / `Config.has_avx2`), not a runtime check —
there is no `caml_has_avx` runtime primitive. Binaries built with AVX
enabled assume the target CPU supports it; select SSE-vs-AVX
implementations when building, not at runtime. On amd64 each
extension has a per-compilation `-f<ext>` / `-fno-<ext>` flag
(`-fpopcnt`, `-flzcnt`, `-fsse3`, `-fssse3`, `-fsse41`, `-fsse42`,
`-fclmul`, `-fbmi`, `-fbmi2`, `-favx`, `-favx2`, `-ff16c`, `-ffma`,
`-favx512f` …); the Haswell-level ones default to what the compiler
was configured with, and the AVX-512 ones are never on by default.

- Requires SSE2 minimum, AVX/AVX2 for 256-bit on x86
- BMI/BMI2 require specific CPU support (Haswell+)
- No auto-vectorization (explicit SIMD required)
- Alignment not automatically guaranteed; align data for better
  load/store performance
- Use unboxed vectors (`#` suffix) in hot loops, process in chunks
  matching the vector width, and put `[@inline always]` on small SIMD
  functions
- Avoid horizontal operations when possible (they're slower)
- Prefer AVX2 for 256-bit operations when available

See also: [SKILL-UNBOXED.md](SKILL-UNBOXED.md) for unboxed types,
[SKILL-ZERO-ALLOC.md](SKILL-ZERO-ALLOC.md) for allocation-free SIMD code.
