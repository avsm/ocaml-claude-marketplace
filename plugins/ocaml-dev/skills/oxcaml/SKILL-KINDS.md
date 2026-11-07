---
name: oxcaml-kinds
description: "OxCaml kind system for classifying types by runtime representation including layouts, kind products, and mode bounds"
---

# OxCaml Kinds: Detailed Guide

Kinds classify types by their runtime representation (layout). They determine
how values are stored, passed, and what operations are valid.

## Kind Hierarchy

```
any                              (* Top kind - any layout *)
├── value_or_null                (* Standard OCaml values, possibly null *)
│   └── value                    (* Standard OCaml boxed values *)
│       └── immediate64          (* Immediate on 64-bit only *)
│           └── immediate        (* Tagged integers, no GC scanning *)
├── void                         (* Zero size — unit# lives here *)
├── float64                      (* Unboxed 64-bit float *)
├── float32                      (* Unboxed 32-bit float *)
├── bits8                        (* Unboxed 8-bit integer; bool# lives here *)
├── bits16                       (* Unboxed 16-bit integer *)
├── bits32                       (* Unboxed 32-bit integer *)
├── bits64                       (* Unboxed 64-bit integer *)
├── word                         (* Native word size *)
├── untagged_immediate           (* int# — native word, untagged *)
└── vec128 / vec256 / vec512     (* SIMD vectors *)
```

Subkinding: a more specific kind can be used where a more general one
is expected (`immediate ≤ immediate64 ≤ value ≤ any`, `float64 ≤ any`,
`bits32 ≤ any`, etc.). Given `val f : ('a : value). 'a -> 'a`, `f 42`
is accepted: `int` is `immediate`, which ≤ `value`.

### What Types Have Which Kinds?

| Kind | Types | GC Scanned | Size |
|------|-------|------------|------|
| `value` | `string`, `'a list`, `'a option`; `float` (but special float array optimization); `int32`, `int64`, `nativeint` (boxed); `int8# array`, `int16# array`, `int# array` (specialized subkind) | Yes | 1 word |
| `immediate` | `int`, `char`, `bool`, `unit` | No | 1 word |
| `float64` | `float#` | No | 8 bytes |
| `float32` | `float32_u` | No | 4 bytes |
| `bits8` | `int8#`, `char#`, `bool#`, `uint8_u` | No | 1 byte |
| `bits16` | `int16#`, `uint16_u` | No | 2 bytes |
| `bits32` | `int32_u`, `uint32_u` | No | 4 bytes |
| `bits64` | `int64_u`, `uint64_u`, block indices `('a, 'b) idx_imm` / `idx_mut` / `idx_atomic` | No | 8 bytes |
| `word` | `nativeint_u`, `unativeint_u` | No | native |
| `untagged_immediate` | `int#` | No | native |
| `void` | `unit#` | - | 0 |
| `vec128` | `int8x16#`, etc. | | |
| `vec256` | `int8x32#`, etc. | | |
| `vec512` | `int8x64#`, etc. (`-extension simd_beta`) | | |
| `any` | (any of above); `'a array#`, `'a iarray#` (type-level only) | Varies | Varies |

`int64_u`, `int32_u`, `nativeint_u` and `float32_u` are predefined
types and the only spelling: the `#`-suffixed names `int64#` etc. no
longer exist. See [SKILL-UNBOXED.md](SKILL-UNBOXED.md).

## Kind Annotations

```ocaml
type t : float64                                (* abstract type *)
type my_float : float64 = float#                (* alias preserving layout *)

(* On type parameters *)
type ('a : value) boxed_list = 'a list
type ('a : float64) float_pair = #('a * 'a)   (* boxed tuples need value *)
type ('a : bits32) container = { value : 'a }
type ('a : value, 'b : float64) mixed = { boxed : 'a; unboxed : 'b }

(* On locally abstract types *)
let process (type a : float64) (x : a) : a = x
let example () = process #3.14  (* a instantiated to float# *)
```

In signatures:

```ocaml
(* Kind-annotated polymorphism *)
val id : ('a : value). 'a -> 'a
val id_float : ('a : float64). 'a -> 'a

(* Mixed — all quantified variables go before the dot *)
val convert : ('a : bits32) ('b : bits64). 'a -> 'b
```

`(type : k)` is an anonymous abstract type of kind `k`, usable where a
type is expected: `type a = (type : float64)`.

### Kind Inference

```ocaml
let f x = x              (* 'a inferred as value (default): 'a -> 'a *)
let g (x : float#) = x   (* inferred from context: float# -> float# *)

(* Flexible: infers kind from usage — x must have kind float64 *)
let f x = (x : float#)
(* Rigid: kind specified at binding — a is exactly float64 *)
let g (type a : float64) (x : a) = x

(* A container constrains its element kind; a function over it
   inherits the constraint: 'a implicitly has kind value *)
type ('a : value) cell = { contents : 'a }
let get (b : 'a cell) : 'a = b.contents
```

---

## Kind Abbreviations

- `value` — boxed OCaml values, GC-managed; defined as
  `value_or_null non_null separable`
- `immediate` — tagged integers: `int`, `char`, `bool`, etc.; it is
  `value non_pointer mod everything`
- `immediate64` — immediate on 64-bit, boxed on 32-bit; it is
  `value non_pointer64 mod global aliased many contended portable
  forkable unyielding immutable stateless external64`
- `mutable_data` — records with mutable fields; it is `value non_float
  mod many portable forkable unyielding stateless` (does not cross
  contention)
- `immutable_data` — immutable non-float data; it is `value non_float
  mod many contended portable forkable unyielding immutable stateless`

### Kind Declarations (`kind_` keyword)

Define a reusable kind alias with `kind_` (the old keyword
`kind_abbrev_` has been renamed — rename it in old code):

```ocaml
kind_ my_int_kind = bits32 mod external_
```

For `compiler-libs` consumers the parsetree constructors are
`Psig_jkind` / `Pstr_jkind`, carrying a `jkind_declaration` record.

### Non-`value` Base Kinds: External, but Not `mod everything`

The base kinds `bits8`, `bits16`, `bits32`, `bits64`, `float32`,
`float64`, `untagged_immediate`, `vec128`, `vec256`, `vec512`, `void`,
`word` **imply externality** (`bits64 mod external_` is just
`bits64`), so write-barrier elision applies automatically. There is no
opt-out: the `_internal` suffix (e.g. `bits64_internal`) no longer
exists, and "non-`value` layout, not `mod external_`" is not
expressible.

They do **not** imply the *modal* axes. An abstract `type t : float64`
does not cross contention or portability; write `float64 mod
everything` for that. (Concrete types such as `float#` and `int64_u`
*do* cross everything — their declared kind is `… mod everything`.)

```ocaml
type t : float64
let f (x : t @ contended) : t @ uncontended = x
(* Error: This value is contended but is expected to be uncontended. *)

type u : float64 mod everything
let g (x : u @ contended) : u @ uncontended = x          (* ok *)
let h (x : float# @ contended) : float# @ uncontended = x  (* ok *)
```

### Scannable Axes

Scannable axes (`non_pointer`, `non_pointer64`, `non_float`,
`separable`, `maybe_separable`, plus nullability) are written after a
layout: `value non_pointer`. A written axis takes the **meet** with
the layout's existing bound — it can only *lower* the axis, never
raise it. `value maybe_separable` therefore equals `value`, and
`k mod axis` is equivalent to `k axis` (prefer the postfix spelling
for scannable axes; `value mod non_float` still parses silently and
prints back as `value non_float`).

Scannable axes can also be written on **abstract kinds** and on
arbitrary kind expressions:

```ocaml
module type S = sig
  kind_ k
  val get : ('a : k separable). 'a array -> 'a
end

type t4 : (value & value) non_pointer     (* parses; warning 184 *)
type t5 : (value mod global) non_pointer
```

### Addressable Kinds

`addressable` is a postfix kind modifier: a kind whose *boxed form can
have a pointer taken into it*. Already addressable: `value` and its
subkinds, `bits64`, `word`, `vec128/256/512`, and products of
addressable kinds. Not addressable: `bits8`, `bits16`, `bits32`,
`float32`, `float64`, `untagged_immediate`, `void`.

```ocaml
type t8 : bits8 addressable
type t : any addressable
kind_ k2 = bits8 addressable
type ('a : any) ptr                 (* your own type *)
val ptr_of : ('a : any addressable). 'a box -> 'a ptr
```

`bits8` and `bits8 addressable` are incomparable; `k addressable` on
an already-addressable kind gives warning 183. Nothing in the stdlib
uses the modifier yet and it does not change runtime representation.

### Kind-Annotation Warnings (all on by default)

| Warning | Fires when | Example |
|---------|------------|---------|
| 181 `imprecise-kind-annotation` | an annotation on a type variable is weaker than what is inferred | `type ('a : any) w = 'a list` — "annotated with kind `any' but was inferred to have kind `value_or_null'" |
| 183 `redundant-kind-modifier` | a scannable-axis modifier is already implied | `value maybe_separable`, `immediate non_null`, `value addressable` |
| 184 `ignored-kind-modifier` | a scannable-axis modifier has no effect on that layout | `float64 non_null`, `(value & value) non_pointer` |
| 211 `redundant-modifier` (was `mod-by-top`) | a `mod` bound is the default for its axis, or implied by another | `value mod local`; `value mod global aliased` ("implied by global") |
| 220 `redundant-modality` | a modality repeats or is overridden on one axis | `val x : int @@ portable portable` |

- 181 hits array code often: `('a : any) … 'a array` is inferred as
  `any separable`, so write `('a : any mod separable)`.
- 211 implications: `global` ⇒ `aliased`, `forkable`, `unyielding`;
  `stateless` ⇒ `portable`; `immutable` ⇒ `contended`. Spelling out
  the full expansion of `mutable_data`/`immutable_data`/`immediate64`
  therefore warns — use the abbreviation.
- The mnemonic `mod-by-top` is **gone**: `[@@@warning "-mod-by-top"]`
  now parses as single-letter warning classes (with an
  `ocaml_deprecated_cli` alert) and silently toggles unrelated
  warnings. Use `-211` (works on both old and new compilers) or
  `-redundant-modifier`.

---

## Kind Products

Unboxed tuples have product kinds:

```ocaml
type pair : float64 & bits32
type triple : float64 & float64 & bits32
type ('a : float64 & bits32) wrapper = { data : 'a }   (* as parameter *)

let p : #(float# * int32_u) = #(#3.14, #42l)   (* kind float64 & bits32 *)
```

---

## Mode Bounds on Kinds

```ocaml
(* Type that crosses locality: a local value may be used as global *)
type t : value mod global

(* Type that is always portable *)
type portable_t : value mod portable

(* Several bounds at once (at most one per axis — two bounds on one
   axis is an error: "The locality axis has already been specified.") *)
type crosser : value mod global portable contended
```

| Bound | Meaning |
|-------|---------|
| `mod global` | Crosses locality (`local` usable as `global`) |
| `mod portable` | Safe for cross-thread |
| `mod external_` | GC ignores it (immediate) |
| `mod immutable` | No mutable fields |

A bound that is the default for its axis (`mod local`, `mod
uncontended`) or implied by another (`mod global aliased`) triggers
warning 211 `redundant-modifier`. `non_float` is a *scannable axis*,
not a mode bound — write it postfix as `value non_float`
(`mod non_float` is an older spelling of the same thing).

---

## Computing Kinds of User Types

```ocaml
(* Records *)
type point = { x : int; y : int }
(* immutable_data with int with int, i.e. immutable_data *)
type counter = { mutable count : int }       (* mutable_data *)
type vec2 = #{ x : float#; y : float# }
(* unboxed record: float64 & float64 mod everything *)

(* Variants *)
type color = Red | Green | Blue              (* all-constant: immediate *)
type 'a option = None | Some of 'a           (* immutable_data with 'a *)
type 'a ref = { mutable contents : 'a }      (* mutable_data with 'a *)

(* Tuples *)
type pair = int * int
(* immutable_data with int with int, i.e. immutable_data *)
type upair = #(int * int)                    (* immediate & immediate *)
```

---

## Implicit Kinds in Signatures and Structures

A floating attribute `[@@@implicit_kind: ...]` declares that specific
type-variable names default to a chosen kind, to avoid repeating the
same `('a : k)` annotation on every declaration. It works in
signatures, in structures and at the module toplevel.

```ocaml
module type S = sig
  [@@@implicit_kind: ('elt : word)]

  type 'elt collection
  val singleton : 'elt -> 'elt collection
  val length : 'elt collection -> int
end

(* Equivalent to: *)
module type S = sig
  type ('elt : word) collection
  val singleton : ('elt : word). 'elt -> 'elt collection
  val length : ('elt : word). 'elt collection -> int
end
```

```ocaml
module M = struct
  [@@@implicit_kind: ('elt : bits64)]
  let f : 'elt -> 'elt array = fun x -> [| x |]
end
(* val f : ('elt : bits64). 'elt -> 'elt array *)
```

Multiple bindings in one attribute use product syntax:

```ocaml
[@@@implicit_kind: ('a : immediate) * ('b : immediate)]

val swap : 'a * 'b -> 'b * 'a
```

Rules and limits:

- **Can't override per-declaration**: a variable declared with an
  implicit kind must have that kind where used. Narrowing or
  contradicting it at a use site is an error.
- **Inherited by nested signatures**: a `module type` or `sig` *within*
  the same module type sees the outer `[@@@implicit_kind: ...]`
  declaration.
- **Re-declaration is allowed**: a later
  `[@@@implicit_kind: ('a : k2)]` shadows an earlier one for
  subsequent items.
- **No propagation through `include`**: if `T` does `include S`, the
  implicit kinds declared in `S` do **not** carry over into `T`.
  Implicit kinds are lexical defaults, not part of the interface.
- **`constraint` clauses**: implicit kinds apply, so a `constraint` on
  `'elt` respects the declared kind. In fact, implicit kinds are
  currently the only way to set a constraint to certain kind values.
- **GADTs**: an implicit `('a : any)` works for a GADT whose
  constructors instantiate `'a` at different layouts.

Use implicit kinds when a signature is dominated by functions taking
one or two named type variables at a non-`value` kind (libraries
specialised to `word`, `bits64`, or an immediate). It is not a
replacement for `ppx_template` for kind-polymorphic code that also
needs a `value` instantiation.

---

## Fields and Constructor Args of Kind `any` (`layouts_beta`)

Under `-extension layouts_beta`, record fields, constructor arguments,
inline-record fields, and unboxed-record fields may have kind `any`.
Projections and constructions require the type to be representable at
the use site:

```ocaml
(* ocamlopt -extension layouts_beta *)
type ('a : any) t = { fst : 'a; mutable snd : 'a }
let a : float# t = { fst = #1.0; snd = #2.0 }
let b : int t = { fst = 1; snd = 2 }
type ('a : any) u = #{ a : 'a; s : string }
let c : int64_u u = #{ a = #1L; s = "" }

let fst (type a : any) (t : a t) = t.fst
(* Error: Record element types must have a representable layout.
          The layout of a is any … *)
```

Without `layouts_beta` the declaration itself is an error (it used to
compile, with the `('a : any)` annotation on a parameter used directly
as a field type *silently narrowed* to `value_or_null`):

```
type ('a : any) t = { fst : 'a }
Error: This construct requires the beta version of the extension
       "layouts", which is disabled and cannot be used
```

The gate fires for record fields, `A of 'a`, inline records, unboxed
records, and fields of an abstract `type t : any`. It does not fire
when the variable only appears under another constructor
(`{ x : 'a list }` is accepted, with warning 181). Fix by dropping the
annotation, writing `('a : value_or_null)`, or enabling
`layouts_beta`.

## Box Types

`'a box` is a predefined type constructor (parameter kind `any`,
result kind `value`) that denotes the boxed version of a type and
reduces during unification. No extension flag needed.

```ocaml
float# box = float        int# box = int
int8# box = int8          bool# box = bool      char# box = char
'a ref# box = 'a ref      int8x16# box = int8x16
#(int * string) box = int * string
string array# box = string array
type r = { x : int }       (* r# box = r *)
type ('a : any) b = 'a box (* float# b = float *)
```

**Not reduced** (left as abstract `value` types): `int64_u box`,
`int32_u box`, `nativeint_u box`, `float32_u box`, `uint8_u box`,
and `u box` for an explicitly-declared unboxed record `u`. In
particular, **`int64_u box` is a different type from `int64`**
(likewise `int32`, `nativeint`, `float32`), and there is no `%box`
primitive yet, so such a type cannot be constructed — box through
`Int64_u.to_int64` etc. instead.

Nominality is preserved (an unboxed record's `box` only unifies with
*its own* boxed record). Float records have no unboxed version and
don't unify with `_ box`. A user type named `box` shadows the
predefined one — avoid that name.

## Layout Polymorphism: `let poly_` / `val poly_` (alpha)

**Requires `-extension layout_poly_alpha`** (neither `layout_poly_beta`
nor `layouts_alpha` enables it; without it: `Error: The extension
"layout_poly" is disabled and cannot be used`).

```ocaml
(* ocamlopt -extension layout_poly_alpha *)
let poly_ id x = x
let poly_ pair x y = #(x, y)

let #(a, b) = pair "a" #1L       (* string, int64_u *)
let c = id #42.0                 (* float# *)

module type S = sig
  val poly_ mk : 'a -> 'b -> #('a * 'b)
  (* only FREE type variables are layout-polymorphic; explicitly
     quantified ones stay value: here only 'b is polymorphic *)
  val poly_ baz : ('a : immediate). 'a -> 'b -> #('a * 'b)
end
```

- `ocamlopt -i` prints `val poly_ id : 'a -> 'a`.
- **Warning 219** (no mnemonic) if a `val poly_` has no
  layout-polymorphic variables — e.g. when every variable is
  explicitly quantified; it then behaves as a plain `val`.
- `let poly_ f = 42` is an **error** ("This binding has no layout
  variables, so poly_ has no effect"; it was warning 217, which no
  longer exists).
- The right-hand side must be a syntactic value (function,
  constructor, tuple, record, constant).
- `let rec poly_` gives warning 218 and is not polymorphic; a
  `let poly_ … and …` group must be all `poly_` or none.
- `layout_` quantifiers are not allowed inside `val poly_`, nor in
  term-level annotations.
- A `poly_` value does not match an ordinary `val id : 'a -> 'a` in a
  signature.
- **Cross-module use is restrictive**: the defining `.mli` must start
  with `@@ static` and the client must see the `.cmx` (`-Ix <dir>`),
  otherwise "The value Lib.pair is dynamic but is expected to be
  static because it is layout-polymorphic and being instantiated
  here."

For production code, `ppx_template` (below) remains the way to write
kind-polymorphic definitions.

---

## Kind Polymorphism with Templates

Since OxCaml doesn't yet have first-class kind polymorphism, use `ppx_template`:

```ocaml
(* Define for multiple kinds *)
let%template id (type a : k) (x : a) : a = x
[@@kind k = (value, float64, bits32, bits64)]

(* Generates: id, id__float64, id__bits32, id__bits64 *)

(* Use with kind attribute *)
let f x = (id [@kind float64]) x
```

Kind-polymorphic types:

```ocaml
type%template ('a : k) cell = { mutable value : 'a }
[@@kind k = (value, float64, bits32)]

let%template get (type a : k) (c : (a cell[@kind k])) : a = c.value
[@@kind k = (value, float64, bits32)]

let%template set (type a : k) (c : (a cell[@kind k])) (v : a) : unit =
  c.value <- v
[@@kind k = (value, float64, bits32)]
```

---

## Kind Errors and Solutions

### "This constant has type float# but an expression was expected of type ('a : value)"

```ocaml
(* ERROR: float# has layout float64, not value ('a. defaults to value) *)
let f : 'a. 'a -> 'a = fun x -> x
let _ = f #3.14

(* FIX: Add kind annotation *)
let f (type a : float64) (x : a) = x
let _ = f #3.14

(* OR use template *)
let%template f (type a : k) (x : a) = x
[@@kind k = (value, float64)]
```

### "This type float# should be an instance of type ('a : value_or_null)"

```ocaml
(* ERROR: list elements must be value *)
let bad : float# list = [#1.0; #2.0]

(* FIX: Box the values *)
let good : float list = [1.0; 2.0]

(* OR use unboxed array *)
let also_good : float# array = [| #1.0; #2.0 |]
```

### "The layout of a is X because... But the layout of a must be a sublayout of Y"

```ocaml
(* ERROR: conflicting kind constraints *)
let bad (type a : value) (x : a) : (_ : float64) = x

(* The error explains: the layout of a is value (from its annotation),
   but it must be a sublayout of float64 *)
```

See also: [SKILL-UNBOXED.md](SKILL-UNBOXED.md) for unboxed types
(including mixed-kind records),
[SKILL-TEMPLATES.md](SKILL-TEMPLATES.md) for kind polymorphism.
