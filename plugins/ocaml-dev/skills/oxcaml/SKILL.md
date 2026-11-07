---
name: oxcaml
description: Working with the OxCaml extensions to OCaml. Use when the oxcaml compiler is available and you need high-performance, unboxing, stack allocation, data-race-free parallelism
---

You are writing code for the OxCaml compiler, a performance-focused fork of
OCaml with Jane Street extensions. This guide covers OxCaml-specific
features only; standard OCaml is assumed.

**Current target**: OxCaml **`5.4.0-ox9`** — `oxcaml-compiler.5.4.0-ox9`
/ `ocaml-variants.5.4.0+ox`, on the **OCaml 5.4** base
(`ocamlopt -version` prints `5.4.0+ox` for every ox release; the exact
check is `opam var oxcaml-compiler:release`). The previous target was
`5.2.0minus-39` (OCaml 5.2 base).

**Migrating a project from minus-39?** Follow
[MIGRATION-39-ox9.md](MIGRATION-39-ox9.md): step-by-step guide,
error-message lookup table, before/after code checked on both
compilers, forms that build on both.

Change history, newest first: [CHANGES-39-ox9.md](CHANGES-39-ox9.md)
(`5.2.0minus-39` → `5.4.0-ox9`: minus-40, the upstream OCaml 5.4
merge, ox1 … ox9), [CHANGES-38-39.md](CHANGES-38-39.md),
[CHANGES-31-38.md](CHANGES-31-38.md),
[CHANGES-25-31.md](CHANGES-25-31.md),
[CHANGES-23-25.md](CHANGES-23-25.md).

### Quick Upgrade Flags (minus-39 → ox9)

When a build fails after a bump from minus-39, check these first
(most common first; the fix for each and an ordered checklist are in
[MIGRATION-39-ox9.md](MIGRATION-39-ox9.md)):

- `int64#`, `int32#`, `nativeint#`, `float32#` are gone → `int64_u`,
  `int32_u`, `nativeint_u`, `float32_u` ("The type int64 has no
  unboxed version. Hint: Did you mean int64_u?"). Literals (`#42L`),
  `float#`/`int8#`/`int#`/…, and `"%…#…"` primitive names are
  unchanged. `int64_u box` ≠ `int64`.
- Rebuild everything, ppx rewriters and tools included (magic numbers
  578 → 586); Jane Street libraries move to `v0.18~preview.133.03+642`.
- `effect` is a keyword → rename identifiers or pass `-keywords 5.2`.
- Tooling gaps: no odoc for ox9 (`dune build @doc` cannot work);
  `ocamlformat 0.29.0+ox` rejects a `version = …` line in
  `.ocamlformat` and cannot parse `| effect` patterns, `overwrite_` or
  `val poly_`; on arm64 `ocaml_simd` needs a `CPATH` workaround to
  install (MIGRATION-39-ox9.md step 0).
- `fst p` / `snd p` on a local pair in tail position is an error:
  bind the result, add `[@nontail]`, or use `exclave_` (section 2).
- `type ('a : any) t = { x : 'a }` needs `-extension layouts_beta`
  (minus-39 silently narrowed the annotation).
- All-void constructors (`A of unit#`) need
  `[@immediate_all_void_constructor]`.
- New default-on warnings: 181 `imprecise-kind-annotation`, 183/184
  kind modifiers, 220 `redundant-modality` (`@@ global aliased`,
  `mutable f : t @@ aliased`); 211 renamed `mod-by-top` →
  `redundant-modifier` (use `-211`); 189 → 75.
- `Effect.perform` raises an `unsafe_effects` alert → use
  `Effect.Safe.perform`.
- `{ t with … }` may not implicitly copy an `[@atomic]` field.
- Integer literals at max+1 (`128s`, `32768S`,
  `9223372036854775808L`, …) are errors — they used to wrap to the
  minimum.
- `compiler-libs.common` was split: add `compiler-libs.frontend` for
  `Path`/`Types`/`Env`/`Typedtree`/….
- SIMD load/store primitives on `float array` removed; AVX-512 types
  need `-extension simd_beta`.
- Runtime 4 retired: `--disable-runtime5` is a configure error.
- Error messages no longer double-quote identifiers in uncoloured
  output (matters when grepping compiler output or comparing against
  older docs) and many were reworded → re-promote expect tests.
- C stubs: `caml_failure_exn` removed (use
  `caml_exception_failure_value`); `caml_callback_res`/`caml_result`
  supersede the `_exn` variants (which still compile).

## Detailed Guides

| Feature | Guide |
|---------|-------|
| **Modes** (local, unique, once, portable, contended) | [SKILL-MODES.md](SKILL-MODES.md) |
| **Stack Allocation** (local_, stack_, exclave_) | [SKILL-STACK-ALLOCATION.md](SKILL-STACK-ALLOCATION.md) |
| **Unboxed Types** (float#, int64_u, mixed blocks) | [SKILL-UNBOXED.md](SKILL-UNBOXED.md) |
| **Kinds** (value, float64, bits32, kind products) | [SKILL-KINDS.md](SKILL-KINDS.md) |
| **Uniqueness** (unique/aliased, once/many) | [SKILL-UNIQUENESS.md](SKILL-UNIQUENESS.md) |
| **Comprehensions** (list/array builders) | [SKILL-COMPREHENSIONS.md](SKILL-COMPREHENSIONS.md) |
| **SIMD** (vector types, SSE/AVX intrinsics) | [SKILL-SIMD.md](SKILL-SIMD.md) |
| **Templates** (ppx_template, mangling) | [SKILL-TEMPLATES.md](SKILL-TEMPLATES.md) |
| **Zero-Alloc** ([@zero_alloc] checking) | [SKILL-ZERO-ALLOC.md](SKILL-ZERO-ALLOC.md) |
| **Base Library** (OxCaml extensions) | [SKILL-BASE.md](SKILL-BASE.md) |
| **Core Library** (OxCaml extensions) | [SKILL-CORE.md](SKILL-CORE.md) |

## 1. Modes

Modes track runtime properties of values. Each mode axis is independent.

| Axis | Values | Default | Purpose |
|------|--------|---------|---------|
| Locality | `local`, `global` | `global` | Where value lives (stack vs heap) |
| Uniqueness | `unique`, `aliased` | `aliased` | Number of references |
| Linearity | `once`, `many` | `many` | How often closures can be called |
| Portability | `portable`, `corruptible`, `shareable`, `nonportable` | `nonportable` | Cross-thread safety |
| Contention | `uncontended`, `shared`, `corrupted`, `contended` | `uncontended` | Thread access patterns |
| Visibility | `read_write`, `read`, `write`, `immutable` | `read_write` | Mutable-field access rights |
| Statefulness | `stateless`, `reading`, `writing`, `stateful` | `stateful` | Closed-over mutable state |

Portability/contention and visibility/statefulness are 4-element
**diamonds** (`shared`/`corrupted` and `read`/`write` are
incomparable). `observing` is now spelled `reading`. This table is the
commonly used subset — there are also `unyielding`/`yielding`,
`forkable`/`unforkable` and `static`/`dynamic` axes. No axis or mode
name changed between minus-39 and ox9.

```ocaml
let f (x @ local) = ...               (* on a parameter *)
let f (x @ local unique once) = ...   (* multiple modes *)
val g : t @ unique once -> t @ aliased many   (* in signatures *)
let x = (expr : t @ local)            (* on an expression *)
let local_ x = ...                    (* local_ is the only legacy let
                                         prefix (global is default) *)
type t = {                            (* record fields: modalities *)
  global_ data : int;                 (* always global *)
  shared : int list @@ aliased;       (* aliased modality *)
  mutable x : int;                    (* mutable => global many aliased
                                         already; writing them out is
                                         warning 220 *)
}
```

Subtyping — a more restrictive mode can be used where a less
restrictive one is expected: `global` ≤ `local`, `unique` ≤ `aliased`,
`many` ≤ `once`, `portable` ≤ `shareable`/`corruptible` ≤
`nonportable`, `uncontended` ≤ `shared`/`corrupted` ≤ `contended`
(diamonds: the two middle modes are incomparable).

[SKILL-MODES.md](SKILL-MODES.md) also covers inference of mode
implications and curry modes, coercions that mode-cross function
returns, and inference-only mode polymorphism (`-extension
mode_polymorphism_alpha`).

## 2. Stack Allocation (Locality)

Stack-allocated values avoid GC overhead but cannot escape their scope
([SKILL-STACK-ALLOCATION.md](SKILL-STACK-ALLOCATION.md)).

```ocaml
let f () =
  let local_ x = (1, 2) in       (* stack-allocated tuple *)
  let p = stack_ (3, 4) in       (* force stack allocation *)
  fst x + snd p

(* Return a local value: allocated in the caller's frame *)
let make_pair x y = exclave_ stack_ (x, y)

(* exclave_ is REQUIRED around a local-returning tail call *)
let g () = exclave_ make_pair 1 2
```

1. Local values cannot escape their defining scope (no storing in
   globals, no returning without `exclave_`).
2. Local values can reference global values; global values cannot
   reference local values.
3. `exclave_` ends the current region so what follows allocates in the
   caller's region; it must be at tail position.
4. A local-returning call in tail position must be under `exclave_`
   ("This local-returning application is in a tail position that is
   not enclosed in an exclave_ expression") — this includes
   `fst p`/`snd p` on a local tuple; bind the result or add
   `[@nontail]` instead if you want a global result.
5. A local value cannot be passed to a tail call ("is local … because
   it is an argument in a tail call") — write `f x [@nontail]`.
6. Stdlib functions (`List.iter`, `Array.fold_left`, …) take *global*
   arguments (and a closure passed to `List.iter` cannot capture a
   local): on local data use explicit recursion
   (`let rec go acc (l @ local) = …`) or Base's local-aware variants.
7. Accumulators can be local refs: `let local_ r = ref 0 in …; !r`.

## 3. Unboxed Types

Unboxed types store values directly without heap allocation. A `#`
suffix or `_u` name means unboxed (`int64#`, `int32#`, `nativeint#`,
`float32#` no longer exist — use the `_u` names); literals use a `#`
prefix, and a minus sign goes outside: `-#1l`.

| Type | Description | Kind | Literal |
|------|-------------|------|---------|
| `float#` | 64-bit float | `float64` | `#3.14` |
| `float32_u` | 32-bit float | `float32` | `#1.0s` |
| `int32_u` | 32-bit int | `bits32` | `#42l` |
| `int64_u` | 64-bit int | `bits64` | `#100L` |
| `nativeint_u` | native int | `word` | `#42n` |
| `int8#` | 8-bit int, untagged | `bits8` | `#42s` |
| `int16#` | 16-bit int, untagged | `bits16` | `#42S` |
| `int#` | native int, untagged | `untagged_immediate` | `#42m` |
| `char#` | 8-bit char, untagged, same layout as `int8#` | `bits8` | `#'x'` |
| `bool#` | 8-bit bool | `bits8` | `#true`, `#false` |
| `unit#` | zero-size | `void` | `#()` |

- `uint8_u`, `uint16_u`, `uint32_u`, `uint64_u`, `unativeint_u` are
  opaque unsigned types: no literals/operations.
- Small numbers also have non-unboxed types: `float32` (`1.0s`),
  `int8` (`42s`), `int16` (`42S`).
- `char#` patterns take ranges: `| #'a'..#'z' -> …`.
- `'a box` is the boxed version of an unboxed type: `float# box =
  float`; `r# box = r` (but `int64_u box <> int64`).

```ocaml
(* Unboxed record - stored inline, not heap-allocated *)
type point = #{ x : float#; y : float# }
let p : point = #{ x = #1.0; y = #2.0 }
let get_x (p : point) = p.#x

(* Unboxed tuple; destructure with a #(...) pattern *)
type pair = #(float# * int32_u)
let first (q : pair) = let #(a, _) = q in a

(* Mixed block: boxed and unboxed fields, unboxed stored flat *)
type mixed = { name : string; value : float#; count : int32_u }

(* or_null: non-allocating option for value types (Null is encoded
   without a box). Predefined: type 'a or_null = Null | This of 'a
   (don't redeclare). The argument must be a non-null value type:
   float# or_null is rejected *)
let find (arr : string array) idx : string or_null =
  if idx < Array.length arr then This arr.(idx) else Null

(* Custom or_null types: any two-constructor variant (one nullary,
   one unary) can opt into the same non-allocating encoding *)
type 'a maybe = Nope | Yep of 'a [@@or_null]
type no_param = A | B of int [@@or_null]   (* payload shape is free *)
```

- Mixed blocks: `[@atomic]` fields (layout `value`) are allowed;
  `[@@flatten_floats]` is optional on records mixing `float` and
  `float#`.
- Arrays of untagged types are packed: `int8# array` and
  `char# array` use 1 byte per element, `int16# array` 2 bytes,
  `int# array` a native word (`[| #0m; #1m |]`). `int8 array` and
  `int16 array` hold tagged immediates: one word each.
- Stdlib's `Array.get`/`set`/`length` only accept `value` elements, so
  `a.(i)` on an unboxed array needs `open Base` (layout-polymorphic
  `Array`) or `[@layout_poly]` externals.
- [SKILL-UNBOXED.md](SKILL-UNBOXED.md) also covers `array#`/`iarray#`,
  block indices (`idx_atomic`, …), all-void constructors and C
  interop.

## 4. Kinds

Kinds classify types by their runtime representation
([SKILL-KINDS.md](SKILL-KINDS.md)). `any` (any layout) is the top.
Under it: `value_or_null` (values, possibly null), which contains
`value` (standard OCaml boxed values), which contains `immediate`
(`int`, `bool`, `char`, …); and, each directly under `any`, `float64`,
`float32`, `bits8`, `bits16`, `bits32`, `bits64`, `word` (native word
size), `untagged_immediate`, `vec128`/`vec256`/`vec512` (SIMD
vectors), `void` (zero-width).

```ocaml
type ('a : float64) container = ...   (* on type parameters *)
val f : ('a : value). 'a -> 'a        (* on type variables in signatures *)
type t : float64                      (* on abstract types *)
type pair : float64 & bits32          (* kind product: unboxed pair of
                                         float# and int32_u *)
type t : value mod global     (* mode bound: crosses locality, a local
                                 t can be used as global *)
type t : value mod portable   (* always portable *)
```

Abbreviations such as `immediate64`, `mutable_data` and
`immutable_data` are defined in SKILL-KINDS.md.

- Kind annotations are linted by default: warning 181 if an annotation
  is weaker than the inferred kind (`('a : any) … 'a list`), 183/184
  for redundant/ignored scannable-axis modifiers, 211 for redundant
  `mod` bounds (`value mod global aliased`, `value mod local`).
- `addressable` is a postfix kind modifier.
- A field of kind `any` needs `-extension layouts_beta`.
- Layout-polymorphic `let poly_`/`val poly_` need `-extension
  layout_poly_alpha`.

## 5. Uniqueness

`unique`: a single reference exists (safe mutation/deallocation);
`aliased`: multiple references may exist
([SKILL-UNIQUENESS.md](SKILL-UNIQUENESS.md)).

```ocaml
val free : t @ unique -> unit            (* unique parameter: consumed *)
val duplicate : t -> t * t @ aliased     (* aliased return *)
val delay_free : t @ unique -> (unit -> unit) @ once
                                         (* once closure: invoked once only *)

(* Aliased modality: container is unique but contents are aliased *)
type 'a aliased_box = { value : 'a @@ aliased } [@@unboxed]

let ok t = match t with Con { field } -> free t
                                         (* OK: match then use uniquely *)
let bad t = match t with                 (* ERROR: using parts twice *)
  | Con { field } ->
    free_field field;                    (* uses field *)
    free t                               (* uses t which contains field *)
let ok t = match t with                  (* OK: different branches *)
  | Con { field } -> if cond then free_field field else free t
```

## 6. Comprehensions

List and array builders
([SKILL-COMPREHENSIONS.md](SKILL-COMPREHENSIONS.md)). Still a beta
extension: compile with **`-extension comprehensions`**.

```ocaml
[ x * 2 for x = 1 to 10 ]
[ x for x = 1 to 100 when x mod 2 = 0 ]             (* filter *)
[ String.uppercase_ascii s for s in strings ]       (* iterate over list *)
[ x + y for x = 1 to 10 for y = 1 to 10 when x < y when x + y < 15 ]
[| f elem for elem in source_array |]               (* array: [| |] *)
[: x for x = 1 to 10 when x mod 2 = 0 :]            (* immutable array *)
[ (x, y) for x = 1 to 3 for y = 1 to 3 ]            (* 9 elements *)
[ (x, y) for x = 1 to 3 and y = 10 to 12 ]          (* 9 elements:
                              [(1,10); (1,11); (1,12); (2,10); ...] *)
```

`for ... for ...` and `for ... and ...` both produce the **Cartesian
product** — comprehensions cannot zip. `for ... for ...` is nested
(inner source re-evaluated each outer iteration; inner iterators may
reference outer variables); with `for ... and ...` sources are
evaluated once upfront and iterators are independent (cannot reference
each other), which enables exact-size array pre-allocation.

## 7. SIMD Vector Types

128-bit and 256-bit SIMD vectors ([SKILL-SIMD.md](SKILL-SIMD.md);
128-bit also works natively on arm64/NEON). 512-bit vectors and `mask`
types exist under `-extension simd_beta` (AVX-512: types and
load/store only — no intrinsics yet). Each type has a boxed and an
unboxed (`#`) form, e.g. `int8x16` / `int8x16#`. 128-bit: `int8x16`,
`int16x8`, `int32x4`, `int64x2`, `float32x4`, `float64x2`; 256-bit:
`int8x32`, `int32x8`, `float64x4`, etc.; 512-bit: `int8x64` …
`float64x8`, `mask`, `mask#`.

```ocaml
(* dune: (libraries ocaml_simd.sse) *)
open Ocaml_simd_sse

let v = Float32x4.set #1.0s #2.0s #3.0s #4.0s   (* float32_u, lane 0 first *)
let v = Float32x4.sqrt v
let #(x, y, z, w) = Float32x4.splat v
let v = Int8x16.String.get text ~byte:0         (* load *)
```

C stubs:

```ocaml
external vec_op : (int8x16[@unboxed]) -> (int8x16[@unboxed]) =
  "boxed_stub" "unboxed_stub"
```

## 8. Templates (ppx_template)

Generate multiple copies of code with different modes/kinds
([SKILL-TEMPLATES.md](SKILL-TEMPLATES.md)).

```ocaml
(* Mode template: generates id (global) and id__local *)
let%template[@mode m = (global, local)] id
  : 'a. 'a @ m -> 'a @ m
  = fun x -> x
(* Instantiate (exclave_ because the local instance returns local) *)
let f (x @ local) = exclave_ (id [@mode local]) x

(* Kind template: generates id (value) and id__float64 *)
let%template[@kind k = (value, float64)] id
  : ('a : k). 'a -> 'a
  = fun x -> x

(* Exclave conditional: local version gets exclave_ (x, y) *)
let%template[@mode m = (global, local)] make_pair x y =
  (x, y) [@exclave_if_local m]
(* ~reasons is needed because (f a, f b) is not a plain allocation of
   identifiers/constants *)
let%template[@mode m = (global, local)] map_pair f (a, b) =
  ((f a, f b) [@exclave_if_local m ~reasons:[ May_return_local ]])

(* Alloc template *)
let%template rec map
  : f:('a -> 'b @ m) -> 'a list -> 'b list @ m
  = fun ~f list ->
    match[@exclave_if_stack a] list with
    | [] -> []
    | hd :: tl -> f hd :: (map [@alloc a]) ~f tl
[@@alloc a @ m = (heap_global, stack_local)]

(* Kind-polymorphic container (not `box` — that name is the predefined
   'a box type) *)
type%template ('a : k) cell = { contents : 'a }
[@@kind k = (value, float64, bits64)]
```

- Portable/nonportable functor variants, short form:
  `module%template.portable Make (M : S) : T`.
- Default floating attributes: `[%%template: [@@@mode.default m =
  (global, local)] val min : t @ m -> t @ m -> t @ m ]`.
- Library functions are instantiated the same way (Base):
  `(List.map [@alloc stack]) lst ~f`,
  `(List.fold [@mode local global]) l ~init ~f`.
- Mangled names follow the `__suffix` pattern.

## 9. Zero-Alloc Checking

Compile-time verification that functions don't allocate.

```ocaml
let[@zero_alloc] fast_add x y = x + y         (* must not allocate *)
let[@zero_alloc] with_local_pair x y =        (* local/stack allocations
                                                 are allowed *)
  let p = stack_ (x, y) in
  fst p + snd p
let[@zero_alloc opt] complex_func x = ...     (* only checked in optimized builds *)
let[@zero_alloc strict] very_strict x = ...   (* no allocation even on error paths *)
let[@zero_alloc assume] external_wrapper x = external_func x
                                              (* trusted to be zero-alloc *)
let[@cold][@zero_alloc assume error] handle_error e = ...
                                              (* assume for error paths *)
val[@zero_alloc] f : int -> int               (* in signatures; also strict *)
val[@zero_alloc arity 2] h : int -> int -> int
[@@@zero_alloc all]                           (* file-level: all functions
                                                 must be zero-alloc *)
let[@zero_alloc ignore] allowed_to_alloc x = [x]   (* opt out *)
```

On failure, `-zero-alloc-checker-details-cutoff -1` prints full
details. See [SKILL-ZERO-ALLOC.md](SKILL-ZERO-ALLOC.md).

## 10. Parallelism & Capsules

Contention and portability modes: section 1 and
[SKILL-MODES.md](SKILL-MODES.md).

**Spawning work.** `Domain.spawn` carries the `do_not_spawn_domains`
and `unsafe_multidomain` alerts. Use the `parallel` library
(`Parallel.fork_join2`, `Parallel_scheduler.with_parallel`; results
are unboxed tuples: `let #(x, y, z) = Parallel.fork_join3 par f1 f2
f3`) for
structured parallelism, or `Multicore.spawn` (compiler `threads`
library) at the low level:

```ocaml
(* dune: (libraries base portable threads) *)
let counter = Portable.Atomic.make 0
let () =
  match Multicore.spawn (fun () -> Portable.Atomic.incr counter) () with
  | Spawned -> ()
  | Failed ((), exn, bt) -> Printexc.raise_with_backtrace exn bt
```

(The opam compiler is built with `--enable-multidomain` and
`--enable-poll-insertion`.)

**Capsules** (`capsule` / `portable` libraries; `Core.Capsule`)
isolate mutable state so it can be shared safely (examples:
[SKILL-CORE.md](SKILL-CORE.md)):

- Read-only sharing — build mutable data, then freeze it:
  `Capsule.Frozen.create (fun () -> …)`, then
  `let (t @ shared) = Capsule.Frozen.unwrap table` and read with
  shared-mode instances such as `(Array.get [@mode shared]) t i`.
- Data tied to the current capsule via an access token:
  `let (P access) = Capsule.Access.current ()`, then
  `Capsule.Data.wrap ~access v` / `Capsule.Data.unwrap ~access d`.
- Locks live in the `await` library
  (`Await.Capsule.Sync.With_mutex.with_lock`); `Capsule.Blocking_sync`
  is deprecated. The low-level key/password API is `Capsule.Prim` (its
  `*_unique`/`*_once` variants are now `[@mode …]` instances).

**Dynamically scoped variables.** `Stdlib.Dynamic` gives fiber-aware,
parallel-safe dynamic binding: `Dynamic.make ()`,
`Dynamic.with_temporarily d v ~f:(fun () -> …)`, and `Dynamic.get d`
returning `Null` or `This v`. Values go in `portable` and are read
`contended portable`; unbound reads `Null`; there is no `set`.
Signatures and example: [CHANGES-39-ox9.md](CHANGES-39-ox9.md).

**Preemption (experimental, opt-in per fiber).**
`Effect.Deep.Preemptible.match_with`/`try_with`, with ticks from
`Domain.Tick.acquire ~interval_usec` or the scoped
`Domain.Tick.with_` — API and example in
[CHANGES-39-ox9.md](CHANGES-39-ox9.md).

**Effect handlers and modes.** The 5.4 syntax is available —
`match c () with v -> v | effect Poke, k -> Effect.Deep.continue k ()`
— with OxCaml caveats:

- a function containing effect cases is forced **nonportable and
  stateful** (you cannot pattern-match on effects inside a `portable`
  or `stateless` function), and variables used anywhere in such a
  `match` (scrutinee or cases) must be global;
- `Effect.perform` raises the **`unsafe_effects` alert**: prefer
  `Effect.Safe.perform h e`, where `h : Effect.Handler.t @ local` is
  handed to the computation by
  `Effect.Deep.Safe.match_with`/`try_with` (or silence with
  `[@@@alert "-unsafe_effects"]`);
- in a file preprocessed by a ppx, `effect` patterns currently fail
  ("migration error: effect patterns is not supported before OCaml
  5.02") — keep handlers in un-preprocessed files or use
  `Effect.Deep.match_with`.

## 11. Borrowing

`borrow_ e` is a prefix expression form that cooperates with the
uniqueness analysis. It is valid in **exactly three positions** —
anywhere else is a hard error ("The borrow_ operator must appear
directly in a valid borrowing context"):

```ocaml
f (borrow_ r)                      (* common: function argument *)
let y = borrow_ r in ...           (* let-binding RHS *)
match borrow_ r with _ -> ...      (* match scrutinee *)
```

Use it where a callee only reads/writes through a unique value
transiently and you want to keep the unique reference after the call.
A borrowed value is `local` + `aliased`, so the callee must accept its
argument `@ local`. Diagnostics: warning 216 `use-during-borrowing`
("This value is used while being borrowed."); error "This value is
used as unique here, but it is being borrowed." (a conflict between a
borrow and a unique use). See [SKILL-MODES.md](SKILL-MODES.md).

## 12. Implicit Kinds in Signatures and Structures

A floating `[@@@implicit_kind: ...]` attribute declares that specific
type-variable names default to a chosen kind, in signatures and in
structures:

```ocaml
module type S = sig
  [@@@implicit_kind: ('elt : word)]
  type 'elt collection                  (* 'elt defaults to kind word *)
  val singleton : 'elt -> 'elt collection
end
```

Multiple names at once:
`[@@@implicit_kind: ('a : immediate) * ('b : immediate)]`.
Implicit kinds can't be overridden by an explicit annotation on the
variable, are inherited by nested signatures (which may re-declare
them), don't propagate through `include`, and apply inside
`constraint` clauses. See [SKILL-KINDS.md](SKILL-KINDS.md).

## 13. Miscellaneous Extensions

**Labeled tuples**

```ocaml
let point = ~x:10, ~y:20              (* create *)
type point = x:int * y:int            (* type *)
let ~x, ~y = point                    (* destructure *)
let get_x (p : x:int * y:int) =       (* partial match needs a type
                                         annotation *)
  let ~x, .. = p in x
val dimensions : image -> width:int * height:int
```

**Immutable arrays**

```ocaml
let arr : string iarray = [: "a"; "b"; "c" :]   (* : instead of | *)
let first = Iarray.get arr 0                    (* Stdlib.Iarray *)
(* [| |] literals and patterns are disambiguated by expected type
   (iarray / floatarray) *)
let arr3 : int iarray = [| 1; 2; 3 |]
(* Covariant - allows safe subtyping *)
let arr2 : obj iarray = (arr : sub_obj iarray :> obj iarray)
```

- `a.:(0)` works only with `Stdlib_stable.IarrayLabels` open — the
  `( .:() )` operator is not in `Stdlib.Iarray` or
  `Stdlib_stable.Iarray`. `IarrayLabels` also keeps the `*_local`
  variants.
- `Stdlib.Iarray` has `equal`, `compare`, `find_index`, `find_mapi`,
  which the narrower `Stdlib_stable.Iarray` shim lacks — beware
  `open Stdlib_stable` shadowing `Iarray`.

**Include functor** — instead of `module T = struct … end`,
`include T`, `include Comparable.Make(T)`:

```ocaml
module M = struct
  type t = ...
  [@@deriving compare, sexp]
  include functor Comparable.Make
end
```

**Let mutable** — restrictions: cannot escape scope, no closure
capture, single variable only.

```ocaml
let triangle n =
  let mutable total = 0 in    (* mutable local variable - no allocation *)
  for i = 1 to n do total <- total + i done;
  total
```

**Polymorphic parameters**

```ocaml
let create (f : 'a. 'a field -> 'a) = { a = f A; b = f B }
val create : ('a. 'a field -> 'a) -> t
```

**Module strengthening** — write `S with M` instead of
`sig type t = M.t end`.

## Debugging Tips

- **Mode errors**: check if you're trying to return local data
  globally. **Kind errors**: ensure type parameters have correct
  layout annotations.
- **Mangled symbols**: `ocamlfilt` (installed with the compiler)
  demangles flat and structured schemes — but only when a line is
  exactly one symbol: `nm x.o | awk '{print $3}' | ocamlfilt`
- **Native debugging**: use the OxCaml LLDB build (`21.1.0+oxcaml1`);
  `b Module.fn` breakpoints and OCaml-syntax value printing work
- Also available (details in [CHANGES-39-ox9.md](CHANGES-39-ox9.md)):
  flags `-open-cmi`, `-functorize`, `-keywords`, `-i-variance`,
  `ocamldep -modules-only`; `Obj.hash_variant`; idle-phase major GC
  (`Xsmall_heap_limit`); C API `caml_region_begin/end`,
  `caml_add_blocks_to_heap`, `OXCAML_RUNTIME`.

## Library Dependencies

- **`Stdlib`** (OCaml 5.4 base): `Iarray`, `Pqueue`, `Pair`, `Repr`,
  `Dynamic` (OxCaml), `Effect.Safe` (OxCaml), `Modes`, plus
  `Dynarray`, `Atomic.Loc`
- **`stdlib_stable`**: `Float32`/`Float32_u`, `Int8`/`Int8_u`,
  `Int16`/`Int16_u`, `Int_u`, `Char_u`, `Or_null`,
  `Idx_imm`/`Idx_mut`/`Idx_atomic`, `Iarray` (narrow compat shim),
  `IarrayLabels` (section 13)
- **`stdlib_upstream_compatible`**: `Float_u`, `Int32_u`, `Int64_u`,
  `Nativeint_u`
- **`threads`**: `Multicore` (`spawn`, `spawn_on`, `max_domains`)
- **`base`**: Jane Street's standard library with comprehensive OxCaml
  mode support — consult [SKILL-BASE.md](SKILL-BASE.md) for
  OxCaml-friendly functions. Mode/alloc instances via ppx_template
  (`(List.map [@alloc stack])`, `(List.fold [@mode local global])`),
  `Modes` (modal wrappers), `Iarray`, layout-polymorphic
  `Array.get`/`set`/`length`
- **`core`**: extended library with I/O and system features — see
  [SKILL-CORE.md](SKILL-CORE.md)
- **`unboxed`**: `Float_u`, `Int32_u`, `Int64_u`, `Nativeint_u`,
  `Float32_u` with full operator sets (`open Unboxed` gives `F64`,
  `F32`, `I32`, `I64`, `Iptr`)
- **`portable`**, **`capsule`**: `Portable.Atomic`, `Capsule`, …
- **`parallel`**, **`await`**, **`concurrent`**: structured
  parallelism, locks, concurrency (opam packages; `parallel` requires
  oxcaml-compiler ≥ 5.4.0-ox9)
- **`ppx_template`**: mode/kind polymorphism via code generation — see
  [SKILL-TEMPLATES.md](SKILL-TEMPLATES.md) for mangling details
- **`ocaml_simd.ppx`** (`ppx_simd`): SIMD shuffle/blend mask generation
- **`ppx_portable`**: `lazy%portable`, `let%portable rec`
- **`ocaml_simd`** (opam package; requires oxcaml-compiler ≥
  5.4.0-ox9), with dune libraries **`ocaml_simd.sse`** (module
  `Ocaml_simd_sse`: 128-bit intrinsics; SSE on amd64, NEON or C stubs
  on arm64) and **`ocaml_simd.avx`** (module `Ocaml_simd_avx`:
  AVX/AVX2 intrinsics, 256-bit; C stubs on arm64). On arm64
  `opam install ocaml_simd` needs the `CPATH` workaround in
  [SKILL-SIMD.md](SKILL-SIMD.md). Opening either module shadows
  `Int32`, `Int64` (and `String` for `Ocaml_simd_sse`).
