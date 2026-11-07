---
name: oxcaml-unboxed
description: "OxCaml unboxed types for heap-free storage including float#, int32_u, int64_u, unboxed records, mixed blocks, and or_null"
---

# OxCaml Unboxed Types: Detailed Guide

Unboxed types store values directly, without heap allocation or pointer
indirection.

## Built-in Unboxed Types

| Boxed | Unboxed | Kind | Size | Literal |
|-------|---------|------|------|---------|
| `float` | `float#` | `float64` | 64-bit | `#3.14` |
| `int32` | `int32_u` | `bits32` | 32-bit | `#42l` |
| `int64` | `int64_u` | `bits64` | 64-bit | `#100L` |
| `nativeint` | `nativeint_u` | `word` | native | `#50n` |
| `float32` | `float32_u` | `float32` | 32-bit | `#1.0s` |
| `int8` | `int8#` | `bits8` | 8-bit | `#42s` |
| `int16` | `int16#` | `bits16` | 16-bit | `#42S` |
| `int` | `int#` | `untagged_immediate` | native | `#42m` (untagged) |
| `char` | `char#` | `bits8` | 8-bit | `#'a'` |
| `bool` | `bool#` | `bits8` | 8-bit | `#true` / `#false` |
| `unit` | `unit#` | `void` | 0 | `#()` |

### `int64#`, `int32#`, `nativeint#`, `float32#` No Longer Exist

Write `int64_u`, `int32_u`, `nativeint_u`, `float32_u`:

```
type t = int64#
         ^^^^^^
Error: The type int64 has no unboxed version.
Hint: Did you mean int64_u?
```

- The alias forms `Int64.t#`, `Int32.t#`, `Nativeint.t#` are errors too
  (with no hint).
- **Literals are unchanged**: `#42L : int64_u`, `#42l : int32_u`,
  `#42n : nativeint_u`, `#1.0s : float32_u`.
- **Only those four were renamed.** `float#`, `int#`, `int8#`, `int16#`,
  `char#`, `bool#`, `unit#`, SIMD types (`int8x16#`…) and record `r#`
  are still spelled with `#`. There is no `float_u`, `int8_u`, `int_u`
  or `char_u` *type* (those are module names only).
- **Library module names are unchanged**; only their `type t` moved:
  `Stdlib_upstream_compatible.{Int32_u,Int64_u,Nativeint_u}.t`,
  `Stdlib_stable.Float32_u.t`. Code that must also build on
  5.2.0minus-39 (which has no `int64_u`) should name the types through
  these modules (`Int64_u.t`, …).
- **`%`-primitive names keep the `#`**: `"%int64#_add"`,
  `"%array_safe_get_indexed_by_int32#"`. (`"%int64_u_add"` is rejected.)
- Boxed `int64` etc. are custom blocks, so they are not the true
  boxed version of the unboxed number: `int64_u box` is **not**
  `int64` (see [SKILL-KINDS.md](SKILL-KINDS.md)).

### Unsigned Unboxed Integers

`uint8_u : bits8`, `uint16_u : bits16`, `uint32_u : bits32`,
`uint64_u : bits64`, `unativeint_u : word` are predefined, opaque types
distinct from the signed ones (part of the default-on `small_numbers`
extension). They have **no literals, no operations and no stdlib
module** — they exist so signedness can be tracked in types (e.g. as
GADT indices). Convert by reinterpretation:

```ocaml
external to_u8 : int8# -> uint8_u = "%identity"
external of_u8 : uint8_u -> int8# = "%identity"
```

Unsigned arithmetic is done on the *signed* types with the unsigned
primitives `_unsigned_div` / `_unsigned_mod`, available for every
integer width, boxed and unboxed:

```ocaml
external udiv64 : int64_u -> int64_u -> int64_u = "%int64#_unsigned_div"
external udiv : int -> int -> int = "%int_unsigned_div"
(* udiv64 (-#1L) #2L = #9223372036854775807L *)
```

### Literals and Conversion from Boxed

```ocaml
open Stdlib_upstream_compatible   (* Float_u, Int32_u, Int64_u, Nativeint_u *)

let x : float# = #3.14159
let z : int64_u = #1_000_000L

(* Negative literals: the minus goes OUTSIDE the # *)
let neg : int32_u = -#1l        (* not #(-1l) — that is a syntax error *)

(* From boxed values *)
let a : float# = Float_u.of_float 3.14
let b : int32_u = Int32_u.of_int32 42l
```

**Literal range**: a decimal literal equal to `max + 1` is an error
for every width, boxed and unboxed (`128s`, `#2147483648l`,
`9223372036854775808L`, `4611686018427387904`); it no longer wraps
silently to the minimum:

```
Error: Integer literal exceeds the range of representable integers of type int64_u
```

Negative minima (`-#128s`) and hex literals filling the width
(`#0xFFs`) are still accepted.

### `bool#` and `unit#`

- `bool#` has kind `bits8 mod everything`. `#true` / `#false` work in
  both expressions and patterns. Use it rather than `int8#` as a
  boolean surrogate — the intent is clearer and the compiler can
  specialise.
- `unit#` has kind `void mod everything` and no runtime
  representation. Typical uses: eliminating a final-continuation
  argument in generic code; filler in kind products where something is
  required but carries no information; modelling C-style `void`
  returns without boxing.

```ocaml
let classify (b : bool#) : int =
  match b with
  | #false -> 0
  | #true  -> 1

type flag = { enabled : bool#; count : int }   (* flat in records *)

let discard (x : unit#) : int = match x with #() -> 0
let noop () : unit# = #()
```

---

## Unboxed Records

```ocaml
open Stdlib_upstream_compatible

type vec3 = #{ x : float#; y : float#; z : float# }   (* syntax: #{ } *)
let v : vec3 = #{ x = #1.0; y = #2.0; z = #3.0 }
let get_x (v : vec3) : float# = v.#x                  (* access: .# *)
let magnitude #{ x; y; z } =                          (* pattern *)
  Float_u.sqrt (
    Float_u.add (Float_u.mul x x)
      (Float_u.add (Float_u.mul y y) (Float_u.mul z z))
  )
```

- No heap allocation for the record itself; records with all unboxed
  fields are stored flat without indirection.
- Passed by value (copied) to functions — building
  `#{ x = …; y = p.#y }` from `p` creates a new record, it does not
  modify `p`.
- Cannot be recursive (no self-references).
- All fields must have a known layout.

Every boxed record `r` also has an implicit unboxed version `r#`
(except float records, `[@@unboxed]` and `[@@flatten_floats]` records,
and records with `[@atomic]` fields).

## Unboxed Tuples

```ocaml
type pair = #(float# * int32_u)                 (* syntax: #( ) *)
let p : #(float# * int32_u) = #(#1.0, #42l)
let #(f, i) = p                                 (* destructure *)
let process : #(float# * float#) -> float# = fun #(a, b) ->
  Stdlib_upstream_compatible.Float_u.add a b
```

---

## Mixed Blocks

Records can mix boxed and unboxed fields; the unboxed fields are
stored inline in the block (after the header):

```ocaml
type particle = {
  name : string;        (* boxed: pointer to string *)
  mass : float#;        (* unboxed, inline: float64, 8 bytes *)
  velocity : float#;    (* unboxed, inline: float64, 8 bytes *)
  charge : int32_u;     (* unboxed, inline: int32, 4 bytes + pad *)
}

let electron =
  { name = "electron"; mass = #9.109e-31; velocity = #0.0; charge = -#1l }
```

The C-visible layout version is **v6**:
`Stdlib_upstream_compatible.mixed_block_layout_v6` on the OCaml side;
C code that asserts on mixed-block layout needs the matching macro
`Assert_mixed_block_layout_v6`. v6 = all-value/void records are
uniform blocks, and all-`float64` records are mixed blocks by default.

### Records Mixing `float` and `float#`

`[@@flatten_floats]` is **optional**. Without it the boxed `float`
field is stored as a pointer and the record gets an unboxed version;
with it the `float` fields are stored flat inline and there is no
`t#`:

```ocaml
type t = { f : float; u : float# }        (* f stored as a pointer *)
type tu = t#                              (* ok *)

type t_flat = { f : float; u : float# } [@@flatten_floats]
(* f stored flat inline;
   t_flat# -> Error: The type t_flat has no unboxed version.
   Hint: Records with [@@flatten_floats] don't get unboxed versions. *)
```

All-`float64` records (`{ x : float#; y : float# }`) are mixed blocks
by default; `[@@represent_as_float_array]` restores the old
float-array representation (errors unless every field is `float64`,
and such records have no `t#`). Both attributes are part of the
representation for module inclusion.

### `[@atomic]` Fields

```ocaml
type c = { mutable n : int [@atomic]; rate : float# }   (* ok *)
let get c = c.n               (* atomic load *)
let set c = c.n <- 3          (* atomic store *)
```

- Atomic fields **must have layout `value`**
  (`{ mutable f : float# [@atomic] }` → `Error: Atomic record fields
  must have layout value.`).
- Atomic `value` fields **are allowed in mixed blocks**.
- Records with `[@atomic]` fields get **no unboxed version** (`t#`).
- `[%atomic.loc r.f]` is forbidden on mixed records and on polymorphic
  fields (`f : 'a. …`).
- **Functional update must not implicitly copy an atomic field.**
  `{ t with y = 2 }` where another field `x` is atomic:

  ```
  Error: Functional updates that implicitly read atomic fields (here x)
         are forbidden. Hint: if you intend to copy the value
         of an atomic field, do so explicitly: { t with x = t.x }
  ```

- Block indices to atomic fields have type `idx_atomic` — see
  "Block Indices" below.

### All-Void Constructors

A variant constructor whose arguments are all void must be annotated:

```ocaml
type t =
  | A of unit# [@immediate_all_void_constructor]
  | C
```

Without the attribute: `Error: All arguments of the constructor A are
void, so it must be annotated with [@immediate_all_void_constructor].`
(Preparation for a representation change from tagged immediate to
empty block.) Nullary and non-void constructors are unaffected.

### Recursive Values Through Mixed Blocks

`let rec` may go through non-value fields of mixed blocks:

```ocaml
type r = { a : #(r option * float#) }
let rec r = { a = #(Some r, #4.0) }
```

---

## Block Indices

A block index is a typed pointer *into* a block. A record-field index
is written `(.field)`; an array-element index has no literal syntax
(`Idx_mut.unsafe_create_into_array`). The types are `('base, 'elt) idx_imm`
(immutable field), `idx_mut` (mutable) and `idx_atomic` (for
`[@atomic]` fields). All have kind `bits64`.
Operations live in `Stdlib_stable.{Idx_imm, Idx_mut, Idx_atomic}`.

```ocaml
type c = { mutable n : int [@atomic]; name : string }
let i : (c, int) idx_atomic = (.n)
let bump c = Stdlib_stable.Idx_atomic.fetch_and_add c i 5
```

`Idx_atomic` provides `get`, `set`, `exchange`, `compare_and_set`,
`compare_exchange`, `fetch_and_add`, `add`, `sub`, `logand`, `logor`,
`logxor`, `incr`, `decr`. Atomic indices work on atomic fields of
mixed records and can deepen through unboxed records (`(.x.#y)`);
there are no atomic array indices.

Indices are not available into float records (records storing `float`
flat), `[@@unboxed]` records, `[@@represent_as_float_array]` records
or `private` records; mutable indices to polymorphic fields are
forbidden.

---

## The `or_null` Type

A non-allocating option for **value** types (`Some x` allocates,
`This x` does not). The argument must be a non-null *value* type —
despite living in the unboxed-types extension, `float# or_null` and
other unboxed arguments are **rejected**. It is predefined, with
constructors `Null` and `This`:

```ocaml
(* predefined:  type 'a or_null = Null | This of 'a   — do NOT
   redeclare it: that would define an ordinary boxed variant *)

(* No allocation: Null is encoded without a box *)
let find (arr : string array) idx : string or_null =
  if idx >= 0 && idx < Array.length arr then
    This arr.(idx)
  else
    Null

(* Pattern match *)
let get_or_default result default =
  match result with
  | Null -> default
  | This x -> x
```

### Custom `[@@or_null]` Types

Any two-constructor variant (one "null" constructor, one unary) can
opt into the same non-allocating null encoding:

```ocaml
type 'a maybe = Nope | Yep of 'a [@@or_null]

(* The payload shape is unconstrained: *)
type no_param = A | B of int [@@or_null]
type ('a, 'b) multi = Nope2 | Yep2 of ('a list * 'b) [@@or_null]
type fn = No_fn | Fn of (unit -> unit) @@ portable [@@or_null]

(* The null constructor may carry a void payload — no
   [@immediate_all_void_constructor] needed here *)
type vn = Nil of unit# | Cons of string [@@or_null]
```

A custom `[@@or_null]` type has the same precise kind as the builtin
(`value_or_null mod everything with <payload>`), so e.g. `int maybe`
mode-crosses like `int or_null`.

Rejected: more than two constructors ("it must have exactly two
constructors"), multi-argument payloads such as `B of int * int`
("each constructor must be nullary or unary"), and GADT constructors.
Re-export the builtin with `type 'a t = 'a or_null
[@@or_null_reexport]` (no constructors written out; mutually exclusive
with `[@@or_null]`).

---

## Unboxed and Untagged Arrays

Arrays of unboxed and untagged types are packed:

```ocaml
let floats : float# array = [| #1.0; #2.0; #3.0 |]
let ints32 : int32_u array = [| #1l; #2l; #3l |]
let ints64 : int64_u array = [| #1L; #2L; #3L |]

(* Untagged small int arrays *)
let bytes : int8# array = [| #0s; #1s; #127s |]
let shorts : int16# array = [| #0S; #1S; #32767S |]
let ints : int# array = [| #0m; #1m; #42m |]   (* int# literal suffix is m *)
let chars : char# array = [| #'a'; #'b'; #'c' |]
```

| Array Type | Bytes per Element | Notes |
|------------|-------------------|-------|
| `float# array` | 8 | Dedicated array tag, packed |
| `float32_u array` | 4 | Dedicated array tag, packed |
| `int64_u array` | 8 | Dedicated array tag, packed |
| `int32_u array` | 4 | Dedicated array tag, packed |
| `int# array` | native word | Untagged, packed |
| `int16# array` | 2 | Untagged, packed |
| `int8# array` | 1 | Untagged, packed |
| `char# array` | 1 | Same as int8# array |

The untagged int arrays use special block tags to encode the exact length
when the element count doesn't fill a whole word.

`'a array#` and `'a iarray#` are the *unboxed versions of the array
types themselves* (kind `any`, type-level only:
`'a array# box = 'a array`). They are not usable as values yet; ignore
them unless working with `box`.

### Accessing Unboxed Arrays

**`Stdlib.Array.get`/`set`/`length` only accept `value` elements**, so
with the plain stdlib `floats.(0)` is an error ("the layout of float#
must be a value layout"). Either `open Base` (whose `Array.get`,
`set`, `length` are layout-polymorphic, so `a.(i)` and
`a.(i) <- v` just work), or declare layout-polymorphic accessors:

```ocaml
module Array = struct
  include Array
  external[@layout_poly] get :
    ('a : any mod separable). 'a array -> int -> 'a = "%array_safe_get"
  external[@layout_poly] set :
    ('a : any mod separable). 'a array -> int -> 'a -> unit
    = "%array_safe_set"
  external[@layout_poly] length :
    ('a : any mod separable). 'a array -> int = "%array_length"
end

let first = floats.(0)        (* now ok: .( ) resolves to Array.get *)
```

(The element kind must be `any mod separable`; a bare `('a : any)` is
rejected for array elements.)

With either in scope (and `open Stdlib_upstream_compatible` for
`Float_u`), prefer `let mutable` over `ref` for unboxed accumulators
and return several results as an unboxed tuple:

```ocaml
let minmax (arr : float# array) : #(float# * float#) =
  let mutable min_val = arr.(0) in
  let mutable max_val = arr.(0) in
  for i = 1 to Array.length arr - 1 do
    let v = arr.(i) in
    if Float_u.compare v min_val < 0 then min_val <- v;
    if Float_u.compare v max_val > 0 then max_val <- v
  done;
  #(min_val, max_val)
```

---

## Operations on Unboxed Types

The operation modules are split across two libraries:

- `Stdlib_upstream_compatible`: `Float_u`, `Int32_u`, `Int64_u`,
  `Nativeint_u`
- `Stdlib_stable`: `Float32_u`, `Int_u`, `Int8_u`, `Int16_u`, `Char_u`
  (and the boxed small numbers `Float32`, `Int8`, `Int16`)

(dune: `(libraries stdlib_stable stdlib_upstream_compatible)`. With
Jane Street libraries, prefer `Float_u`, `Int64_u`, … from the
`unboxed` package — `open Unboxed`; `Core` does not provide them.)

```ocaml
open Stdlib_upstream_compatible

let compute x y =
  Float_u.div (Float_u.sqrt (Float_u.add x y)) (Float_u.mul x y)
let diff x y = Float_u.sub x y
let hash x y = Int32_u.add (Int32_u.mul x #31l) y

(* To/from boxed *)
let box (x : float#) : float = Float_u.to_float x
let unbox (x : float) : float# = Float_u.of_float x

(* Between sizes *)
let widen (x : int32_u) : int64_u = Int64_u.of_int32 (Int32_u.to_int32 x)
```

Unboxed types have non-`value` kinds (table above). Kind annotations
(`type ('a : float64) t = { f : 'a }`,
`val process : ('a : float64). 'a -> 'a`) and fields of kind `any`
(which require `-extension layouts_beta`) are covered in
[SKILL-KINDS.md](SKILL-KINDS.md).

---

## C Interop

```ocaml
(* Unboxed externals - no boxing overhead. Use EITHER per-argument
   [@unboxed] OR the declaration-level [@@unboxed], never both. *)
external sin : (float[@unboxed]) -> (float[@unboxed]) =
  "caml_sin_float" "sin" [@@noalloc]

(* With float# directly *)
external fast_sin : float# -> float# =
  "boxed_sin" "unboxed_sin"
```

```c
#include <math.h>
#include <caml/mlvalues.h>
#include <caml/alloc.h>

// Unboxed version - called directly
double unboxed_sin(double x) { return sin(x); }

// Boxed wrapper
CAMLprim value boxed_sin(value v) {
  return caml_copy_double(unboxed_sin(Double_val(v)));
}
```

`#ifdef OXCAML_RUNTIME` (defined to `1` in `caml/config.h`) lets a
stub file distinguish OxCaml from stock OCaml.

### Small-Int Bit Intrinsics

`ctz`, `clz`, and `popcnt` are available for `int8#` and `int16#`
as `[@@builtin]` externals named
`caml_int8_clz_untagged_to_untagged`,
`caml_int16_popcnt_untagged_to_untagged`, etc. (all targets). The
amd64 backend additionally recognises these `[@@builtin]` names
(rejected on arm64): `caml_popcnt_int16`, `caml_lzcnt_int16`,
`caml_bmi_tzcnt_int16`.

### Small-Int Indexing

Arrays, strings, bigstrings, and bytes can store `int8#`/`int16#`
values and be indexed by `int8#` / `int16#`. Primitives:

- `%caml_bytes_geti8`, `%caml_bytes_geti16` (sign-extending reads)
- `set8` / `set16` family, with `#` (untagged) and
  `_indexed_by_int8#` / `_indexed_by_int16#` variants

---

## Pitfalls

### Unboxed Values and Polymorphism

```ocaml
(* An un-annotated function is NOT layout-polymorphic: its layout is
   fixed by the first use *)
let id x = x
let _ = id #3.14        (* ok — id : ('a : float64). 'a -> 'a *)
let _ = id 3            (* ERROR: The layout of int must be a
                           sublayout of float64 *)

(* An explicitly polymorphic annotation defaults 'a to kind value *)
let id2 : 'a. 'a -> 'a = fun x -> x
let _ = id2 #3.14       (* ERROR: float# is not a value *)

(* FIX: one function per layout (annotate), or use ppx_template *)
let id_f64 (type a : float64) (x : a) = x
```

### Cannot Store Unboxed in Option — or in or_null

A nullable *unboxed* result needs a validity flag or sentinel instead:

```ocaml
(* ERROR: option expects value kind *)
let bad : float# option = Some #3.14

(* ALSO AN ERROR: or_null expects value kind too *)
let bad2 : float# or_null = This #3.14

(* FIX: box at the boundary, or use an unboxed tuple with a flag *)
let good : float option = Some (Float_u.to_float #3.14)
let good2 : #(bool * float#) = #(true, #3.14)
```

### Partial GADT Matches Cannot Fix a Parameter's Layout

A function parameter whose layout is only known from a
*non-exhaustive* GADT match on an earlier parameter is an error
(it used to be accepted with warning 8 and could segfault):

```ocaml
type ('a : any) t = V : 'a t | B : ('a : bits64) t
let f : type (a : any). a t -> a -> unit -> a = fun V x () -> x
(* Error: This function's argument has type a, whose layout is known
   only from the GADT pattern match at … That match is not exhaustive,
   so a caller could reach this argument at a different layout. *)
```

Make the match exhaustive, or move it inside the function body.

See also: [SKILL-KINDS.md](SKILL-KINDS.md) for the kind system,
[SKILL-ZERO-ALLOC.md](SKILL-ZERO-ALLOC.md) for allocation-free code.
