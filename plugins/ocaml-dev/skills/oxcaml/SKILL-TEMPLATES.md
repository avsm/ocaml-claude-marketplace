---
name: oxcaml-templates
description: "OxCaml ppx_template for generating mode-polymorphic and kind-polymorphic code variants with name mangling"
---

# OxCaml Templates (ppx_template): Detailed Guide

`ppx_template` generates multiple compile-time copies of code with
different modes, kinds, or modalities. For `ppx_template
v0.18~preview.133.03+642` (OxCaml 5.4.0-ox9). Enable it with
`(preprocess (pps ppx_template))` — or `ppx_jane`, which includes it.

Two kinds of attribute:

- **poly** attributes *define* instances: `[@mode m = (global, local)]`,
  `[@@kind k = (value, float64)]`. They only work inside a `%template`
  node — outside one they are silently left in place and do nothing.
- **mono** attributes *select* an instance: `(id [@mode local])`,
  `Float [@kind float64]`. They work **anywhere** in a file preprocessed
  by the ppx; no enclosing `%template` is needed (except `[@synchro …]`,
  which is silently ignored outside a `%template` node).

## Extension Points

| Form | Syntax |
|------|--------|
| On an item | `let%template …`, `val%template …`, `type%template …`, `module%template …` (also module type / include / open) |
| Structure block | `[%%template let f x = x  let g x = x + 1 ]` |
| Signature block (note colon) | `[%%template: val f : 'a -> 'a  val g : int -> int ]` |
| Expression | `let x = [%template fun x -> x]` |

## Mode Templates

```ocaml
let%template[@mode m = (global, local)] identity
  : 'a @ m -> 'a @ m
  = fun x -> x

(* Generates:
   val identity : 'a -> 'a
   val identity__local : 'a @ local -> 'a @ local
*)
```

Instantiating:

```ocaml
(* Mono attribute — works anywhere, no %template node needed.
   (exclave_ because the local instance returns a local value) *)
let f (x @ local) = exclave_ (identity [@mode local]) x

(* Or using the mangled name directly (not recommended) *)
let f2 (x @ local) = exclave_ identity__local x
```

Multiple mode variables:

```ocaml
let%template[@mode m1 = (global, local), m2 = (portable, nonportable)]
  process (x @ m1) (y @ m2) = ()

(* Generates 2 × 2 = 4 versions:
   process, process__local, process__portable, process__local__portable
   (defaults — global, nonportable — contribute no suffix) *)
```

A single function may also take several values on the *same* axis; the
instance is then named by all of them in order, e.g. Base's
`List.fold__local__global` is selected by
`(List.fold [@mode local global])`.

### The exclave_if_local Attribute

Conditionally add `exclave_` for local returns:

```ocaml
let%template[@mode m = (global, local)] make_pair x y =
  (x, y) [@exclave_if_local m]

(* global version: just returns (x, y) *)
(* local version: exclave_ (x, y) *)
```

**Restrictions**: without further annotation it is only valid on:
- Pure syntactic allocations (tuples, records, …) built entirely from
  identifiers, record fields and constants
- Tailcalls

Any other expression needs an explicit justification via `~reasons`
(`May_return_local`, `May_return_regional`, `Will_return_unboxed`):

```ocaml
let%template[@mode m = (global, local)] map_pair f (a, b) =
  ((f a, f b) [@exclave_if_local m ~reasons:[ May_return_local ]])
```

Without it: `[%template]: exclave_if_local is only allowed on:
tailcalls; syntactic allocations (e.g. tuples) consisting entirely of
identifiers, record fields, and/or constants; or when given a nonempty
~reasons argument.`

## Kind Templates

```ocaml
let%template[@kind k = (value, float64, bits32, bits64)]
  id (type a : k) (x : a) : a = x

(* Generates:
   val id : 'a -> 'a
   val id__float64 : ('a : float64). 'a -> 'a
   val id__bits32 : ('a : bits32). 'a -> 'a
   val id__bits64 : ('a : bits64). 'a -> 'a
*)
```

Kind products, and kinds with a mode modifier:

```ocaml
let%template[@kind k = (value, value & value, (value & value) & value)]
  wrap (type a : k) (x : a) = x
(* Generates: wrap, wrap__'value_value', wrap__''value_value'_value' *)

let%template[@kind k = (value, value mod portable)]
  f (type a : k) (x : a) = x
(* Generates: f, f__'value_mod_portable' *)
```

Use locally abstract types for kind polymorphism:

```ocaml
(* ERROR: type variables unify across the instances of one block *)
let%template f (x : ('a : k)) : 'a = x
[@@kind k = (value, float64)]

(* CORRECT: locally abstract types are independent *)
let%template f (type a : k) (x : a) : a = x
[@@kind k = (value, float64)]
```

## Alloc Templates

For ad-hoc polymorphism over allocation behavior (heap vs stack):

```ocaml
let%template[@alloc a @ m = (heap_global, stack_local)]
  make_pair x y =
    (x, y) [@exclave_if_stack a]

(* heap instance   make_pair        : (x, y) *)
(* stack instance  make_pair__stack : exclave_ (x, y) *)
```

`a` is the alloc variable (`heap` / `stack`) and `m` the matching mode
(`global` / `local`); only `a` is used for mangling. The pair can also
be spelled `(heap @ global, stack @ local)`, and libraries that don't
need to name the alloc variable write `[@@alloc __ @ m = …]`.

The alloc variable alone:

```ocaml
let%template[@alloc a = (heap, stack)]
  f x = x [@exclave_if_stack a]
```

With `zero_alloc_if_stack`:

```ocaml
let%template[@alloc a = (heap, stack)]
  process x = compute x
[@@zero_alloc_if_stack a]

(* process__stack gets the [@zero_alloc] attribute *)
```

`[@exclave_if_stack a]` may also be put on a `match`
(`match[@exclave_if_stack a] t with …`).

## Modality Templates

For portable/nonportable functor patterns:

```ocaml
module%template[@modality p = (nonportable, portable)]
  Make (M : sig @@ p include S end) : sig @@ p include T end
  = struct let y = M.x end

(* Generates Make and Make__portable *)
```

### Shorthand: %template.portable

```ocaml
(* Long form *)
module%template[@modality p = (nonportable, portable)]
  F (M : sig include S @@ p end) : sig include T @@ p end = ...

(* Short form *)
module%template.portable F (M : S) : T = ...

(* Select the portable instance (all three are the same module) *)
module A = F [@modality portable]
module B = F [@mode portable]
module C = F__portable
```

To forward the choice to other portable functors inside the body, name
the modality variable:

```ocaml
open Core

module%template.portable [@modality p] Make_key
    (T : sig type t [@@deriving compare, sexp, hash] end) = struct
  include T
  include Comparable.Make [@modality p] (T)
  include Hashable.Make [@modality p] (T)
end

module Int_key = Make_key [@modality portable] (Int)
```

`%template.portable` only makes sense on a **functor**; on a plain
`struct … end` it does not select anything — it just emits two
identical copies.

## Floating Attributes

Apply to all subsequent items in scope:

```ocaml
[%%template:
[@@@mode.default m = (global, local)]

val min : t @ m -> t @ m -> t @ m
val max : t @ m -> t @ m -> t @ m
val clamp : t @ m -> low:t @ m -> high:t @ m -> t @ m
]

(* All three functions get both global and local versions *)
```

Available: `[@@@mode.default ...]`, `[@@@kind.default ...]`,
`[@@@modality.default ...]`, `[@@@alloc.default ...]`. Without
`.default`, names aren't mangled:

```ocaml
[@@@mode m = (global, local)]  (* No mangling *)
[@@@mode.default m = (global, local)]  (* With mangling *)
```

## Name Mangling

**WARNING: Mangling scheme may change. Don't rely on specific names —
prefer the mono attributes (`(f [@kind bits64] [@mode local])`).**

```
base_name[__kind_suffix][__mode_suffix][__modality_suffix][__alloc_suffix]
```

The axes are always emitted in that order (kind, mode, modality,
alloc), e.g. `d__bits64__local__portable__stack`. Within an axis,
values appear in the order written. Product kinds and `mod` kinds are
grouped with single quotes; nested products nest the quotes
(`__''value_value'_value'` for `(value & value) & value`).

| Axis | Default (no suffix) |
|------|---------------------|
| Kind | `value` (and `value_or_null`) |
| Mode | `global`, `nonportable`, `uncontended`, `aliased`, `many`, `unyielding`, `forkable`, `read_write`, `stateful` |
| Modality | `local`, `nonportable`, `uncontended`, `unique`, `once`, `yielding`, `unforkable`, `read_write`, `stateful` |
| Alloc | `heap` |

Because `value_or_null` is also a default,
`[@@kind k = (value, value_or_null)]` produces two unsuffixed
instances and fails ("Variable f is bound several times…" on a `let`,
"Multiple definition…" on a type or module) — use `.explicit` (below)
for that.

### Explicit Mangling: `.explicit`

Give *every* instance a suffix, including the defaults:

```ocaml
let%template id (type a : k) (x : a) = x
[@@kind.explicit k = (value, bits64)]
(* id__value, id__bits64 — no plain `id` *)
```

Instances defined this way must also be *requested* with `.explicit`:
`(id [@kind.explicit value])`. The ordinary `(id [@kind value])`
drops the default and looks for plain `id`, which does not exist
(`Unbound value id`). The same holds for modes — e.g.
`Capsule.Prim.Key.box` only exists as
`(Key.box [@mode.explicit global unique])` and
`(Key.box [@mode.explicit global aliased])`.

Also: `.explicit_plus_unmangled`, `[@@@kind.default.explicit …]`,
`[@@@kind.default_if_multiple …]`.

### Punning

Shorthand when you just want mangling without a new binding. Poly
attributes need an enclosing `%template` node:

```ocaml
[%%template
[@@@kind k = (value, bits64)]

(* Long form *)
let f x = x [@@kind k = k]

(* Punned form *)
let g x = x [@@kind k]
]
(* f, f__bits64, g, g__bits64 *)

(* For monomorphic instances *)
let%template h x = x [@@kind float64]  (* Just mangles to h__float64 *)
```

## Axes

`kind`, `kind_set`, `mode`, `modality`, `alloc`, `synchro`. A variable
name can't be reused across axes (or across mode sub-axes).

- **Modes accepted**: `global`/`local`, `nonportable`/`shareable`/
  `portable`, `uncontended`/`shared`/`contended`, `aliased`/`unique`,
  `many`/`once`, `unyielding`/`yielding`, `forkable`/`unforkable`,
  `read_write`/`read`/`write`/`immutable`,
  `stateful`/`reading`/`writing`/`stateless`, `corrupted`, `corruptible`.
- **Kinds accepted**: `value`, `value_or_null`, `void`, `word`,
  `bits8`/`16`/`32`/`64`, `float32`/`64`, `immediate`, `immediate64`,
  `vec128`/`256`, `untagged_immediate`, `mutable_data`,
  `immutable_data`, products `k1 & k2`, and `k mod …`.

### Kind Sets

Name a set of kinds once and reuse it:

```ocaml
[%%template
[@@@kind_set.define all = (value, bits64)]

module type [@kind_set ks = (value, all)] Monad = sig
  type ('a : k) t [@@kind k = ks]
end]

module type M = Monad [@kind_set all]    (* = Monad__''all'' *)
```

Built-in kind sets: `base_non_value = (bits64, bits32, word, float64,
float32)`, `base` (= `value` + `base_non_value`), and the variants
`value_with_imm`, `value_or_null_with_imm`, `base_with_imm`,
`base_or_null`, `base_or_null_with_imm`.

### Kind Coercion with `or`

Inside a template, pick an instance at `k1` *or* fall back to another
kind when `k1` is the default:

```ocaml
let%template foo (type a : k1) (_x : a) = ()
[@@kind k1 = (value_or_null, float64)]

let%template bar (type a : k1) (x : a) = (foo [@kind (k1 or value_or_null)]) x
[@@kind k1 = (value, float64)]
```

### Correlated Variables (Tuples)

Template two variables together rather than over their product; the
instance is named by the first component:

```ocaml
module%template Delay = struct
  let return x () = x
end
[@@mode (p, c) = ((nonportable, uncontended), (portable, contended))]

module D = Delay [@mode portable]        (* Delay__portable *)
```

### Synchro

The `synchro` axis (`unsync` — the default — or `sync`) abstracts over
whether an operation needs synchronisation, usually paired with a
contention mode:

```ocaml
let%template[@synchro s @ c = (unsync_uncontended, sync_shared)]
  rd (r : int ref @ c) = ()
(* rd, rd__sync *)
```

(Tuples available: `unsync_uncontended`, `sync_shared`,
`sync_contended`.) This is where Base names such as
`Hashtbl.iter__sync` and `Hashtbl.fold__sync` come from
(`Hashtbl.find__shared` is a plain `[@mode shared]` instance). Select
them inside a template node: `[%template (Hashtbl.iter [@synchro sync])]`.

## Common Patterns

Most of the time you only *select* instances that Base/Core define —
`(List.map [@alloc stack])`, `(List.fold [@mode local global])`,
`Comparable.Make [@mode portable] (T)`; see
[SKILL-BASE.md](SKILL-BASE.md).

### Layout-Polymorphic Float Module

`Float_u` (unboxed `float#` operations) comes from the `unboxed`
package: `(libraries core unboxed)`.

```ocaml
open Core

open [%template
  module [@kind value] Float = Float
  module [@kind float64] Float = Float_u
]

(* Float [@kind value] is Core's Float; Float [@kind float64] is Float_u *)
module%template[@kind k = (value, float64)] Arith : sig
  type t : k
  val add : t -> t -> t
  val sub : t -> t -> t
end = Float [@kind k]
(* Arith : value arithmetic;  Arith__float64 : float# arithmetic *)
```

Continuing with the `open [%template …]` aliases above — include with
mangling:

```ocaml
include%template[@kind k = (value, float64)] struct
  open Float [@kind k]

  let[@kind k] double x = add x x
  let[@kind k] zero x = sub x x
end

let a = double 1.0
let b = double__float64 #1.0
```

A module templated over kinds:

```ocaml
module%template[@kind k = (value, float64)] Make = struct
  open Float [@kind k]

  let[@kind k] process = add
end

include%template Make [@kind value]
include%template Make [@kind float64]

let a = process 1.0 2.0
let b = process__float64 #1.0 #2.0
```

### Recursive Functions

`let%template rec` expands to one `let rec … and …` group across all
instances, which the compiler optimises poorly. Prefer an inner
`let rec`:

```ocaml
let%template map t ~f =
  (let rec map t =
     match[@exclave_if_stack a] t with
     | [] -> []
     | hd :: tl -> f hd :: map tl
   in
   map t [@nontail]) [@exclave_if_stack a]
[@@alloc a = (heap, stack)]
```

### Type and Module Substitutions

Destructive substitutions can be templated; the mangled names stay
visible after the `[%%template: …]` block:

```ocaml
module type%template A = sig
  [@@@kind.default k = (bits64, float64)]

  type ('a : k) t := 'a
  type nonrec 'a t = 'a t [@kind k]
end

module type B = sig
  [%%template:
  [@@@kind.default k = (bits64, float64)]
  type ('a : k) t := 'a]

  val f : ('a : bits64). 'a t__bits64 -> 'a
  val g : ('a : float64). ('a t [@kind float64]) -> 'a
end
```

## Debugging

- `dune describe pp path/to/file.ml` shows what templates expand to.
- If unsure of mangling, test with explicit instantiation:
  `let test = (some_function [@kind float64] [@mode local])`
  (= `some_function__float64__local`). If it compiles, the instance
  exists.
- **Effect syntax under a ppx**: `match … with effect E, k -> …`
  (OCaml 5.3+ syntax) currently **fails in any file preprocessed by a
  ppx**, including `ppx_template`: `Error: migration error: effect
  patterns is not supported before OCaml 5.02`. Keep effect handlers
  in a file without `(preprocess …)`, or use the `Effect.Deep`
  function API.

## Limitations

- All instances are generated eagerly, used or not (no SFINAE), so
  code size increases with the number of instances. Expansion is at
  compile time — no runtime overhead — and each instance is fully
  optimized independently.
- No dependent types - can't compute kinds from values
- Right-nested kind products written directly are flattened:
  `value & (value & value)` is treated as `value & value & value`
  (`f__'value_value_value'`)

See also: [SKILL-MODES.md](SKILL-MODES.md) for mode details,
[SKILL-KINDS.md](SKILL-KINDS.md) for kind details,
[SKILL-BASE.md](SKILL-BASE.md) for the instances Base provides.
