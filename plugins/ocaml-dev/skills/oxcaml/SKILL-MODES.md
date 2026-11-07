---
name: oxcaml-modes
description: "OxCaml modal types for locality, uniqueness, linearity, portability, and contention tracking"
---

# OxCaml Modes: Detailed Guide

Modes are compile-time properties of values (no runtime overhead).

## Mode Axes

| Axis | Modes, least → greatest | Default | Tracks |
|------|-------------------------|---------|--------|
| locality | `global` < `local` | `global` | Stack vs heap allocation |
| forkable | `forkable` < `unforkable` | `forkable` | — |
| yielding | `unyielding` < `yielding` | `unyielding` | May the value perform effects (see "Effects and the Yielding Axis") |
| linearity | `many` < `once` | `many` | Closure invocation count |
| uniqueness | `unique` < `aliased` | `aliased` | Single vs multiple references |
| statefulness | `stateless` < `reading`, `writing` < `stateful` | `stateful` | Closed-over mutable state |
| visibility | `read_write` < `read`, `write` < `immutable` | `read_write` | Mutable-field access rights |
| portability | `portable` < `shareable`, `corruptible` < `nonportable` | `nonportable` | Cross-thread safety |
| contention | `uncontended` < `shared`, `corrupted` < `contended` | `uncontended` | Concurrent access |
| staticity | `static` < `dynamic` | `dynamic` | Compile-time computability; used by layout polymorphism and metaprogramming |

- Axes are independent: a value can be `local unique once` or
  `global aliased many`. All ten are writable; forkable, yielding and
  staticity are rarely needed directly.
- **Subtyping**: a value at a lesser mode can be used where a greater
  one is expected (`global` as `local`, `unique` as `aliased`, `many`
  as `once`); the reverse is a type error.
- The four-mode axes are **diamonds**: the two middle modes are
  incomparable (`shared`/`corrupted` between `uncontended` and
  `contended`, `read`/`write` between `read_write` and `immutable`,
  likewise `shareable`/`corruptible` and `reading`/`writing`).
- `reading` was formerly spelled `observing`.
- Printed types list modes in the table's order (`@ local once unique`,
  where older compilers printed `@ local unique once`), and `static`
  is printed — re-promote expect tests.

### Portability and Contention

| Mode | Meaning |
|------|---------|
| `nonportable` | Functions capturing uncontended mutable state; cannot escape current thread |
| `shareable` | Functions capturing shared state; may execute in parallel |
| `corruptible` | Functions closing over only corrupted values |
| `portable` | Functions capturing all values at contended; may execute concurrently |
| `uncontended` | No other thread has access; mutable parts may be read and written |
| `shared` | Several threads may read; unprotected mutable parts are read-only |
| `corrupted` | Several threads may write (but not read); unprotected mutable parts are write-only |
| `contended` | Another thread may read and write; unprotected mutable parts cannot be accessed |

### Visibility and Statefulness

`write` visibility permits *write-only* access to mutable fields;
`writing` closures may write but not read closed-over mutable state.

```ocaml
let mostly_const : int ref @ write -> unit = fun r ->
  r.contents <- 0      (* allowed: write is permitted *)
  (* r.contents (a read) — error: "This value is write but is expected
     to be read or read_write because its mutable field contents is
     being read."  Note `r := 0` is also rejected: `:=` is an ordinary
     function expecting a read_write ref. *)
```

---

## Syntax

| Location | Syntax | Example |
|----------|--------|---------|
| Parameter | `(x @ mode)` | `(x @ local unique)` |
| Signature arrow | `t @ mode ->` | `t @ local -> t @ global` |
| Expression | `(e : t @ mode)` | `(x : int @ local)` |
| Let binding | `let x @ mode =` / `let x : t @ mode =` | `let x @ local = ...` |
| Let binding (legacy) | `let local_ x =` (only `local_`) | `let local_ x = ...` |
| Function result | `let f x @ mode =` / `let f x : t @ mode =` | `let f x @ local = ...` |
| Mode-only constraint | `(e : @ mode)` | `(x : @ aliased)` |
| Record field (legacy) | `global_ field :` (only `global_`) | `global_ data : string` |
| Record field | `: t @@ modality` | `: t @@ aliased` |

- The type comes **first**, then the mode:
  `let process (data : int array @ local) = ...`;
  `(data @ local : int array)` is a syntax error.
- An expression annotation can cast to a weaker mode:
  `(unique_val : t @ aliased)`.
- `local_` is the **only** legacy let-binding prefix. `stack_` is an
  expression operator, not a let prefix (`let z = stack_ (5, 6)`).
  `global` is the default; there is no `let global_` form (`global_`
  exists only as a record-field modality prefix).
- **Removed**: the legacy `unique_` and `once_` prefixes are no longer
  keywords (`unique_ e` is "Unbound value unique_"; `let unique_ x = e`
  silently defines a function named `unique_`). Use `(x @ unique)` in
  patterns and `(e : @ unique)` on expressions.

In signatures, parenthesise a function type to give it a mode:

```ocaml
val f : 'a @ local unique -> 'a @ global aliased
val compose : ('a -> 'b) @ once -> ('b -> 'c) @ once -> ('a -> 'c) @ once
val run : (unit -> 'a @ portable) @ portable -> 'a
```

### Modalities (Record Fields)

Modalities specify the mode of field contents relative to the record:

```ocaml
type t = {
  global_ name : string;           (* always global, even if record is local *)
  shared_data : int list @@ aliased; (* always aliased *)
  callback : (unit -> unit) @@ many; (* always many, not once *)
  mutable count : int;             (* mutable fields are implicitly
                                      @@ global many aliased *)
}
```

`@@ global` always implies `@@ aliased` (and `forkable`,
`unyielding`); `@@ global unique` is a hard error ("The modality
global can't be used together with unique") — this ensures soundness
of borrowing.

**Warning 220 `redundant-modality`** (on by default) fires on any
modality that changes nothing — so do *not* write the implied ones
out:

```ocaml
type t = { a : int ref @@ global aliased }   (* warning: aliased implied *)
type u = { mutable n : int ref @@ aliased }  (* warning: implied by mutable *)
type v = { global_ y : string @@ global }    (* warning *)
module type S = sig
  val y : int ref @@ stateless portable      (* warning: portable implied *)
  val z : int ref @@ nonportable             (* warning: that's the default *)
end
```

Write `@@ global`, and leave `mutable` fields bare. Silence with
`-w -220` if you must.

---

## Deep Modes

Modes are **deep**: if a tuple is local, all its components are local;
if a record is unique, all its (non-modality) fields are unique
(per-field tracking: [SKILL-UNIQUENESS.md](SKILL-UNIQUENESS.md)).
Modalities break depth:

```ocaml
type container = {
  global_ data : string;  (* data is always global *)
  local_stuff : int list; (* follows container's mode *)
}

let f (c @ local) =
  let s = c.data in   (* s is global! *)
  let l = c.local_stuff in  (* l is local *)
  s  (* can return s *)
```

---

## Mode Inference

Unannotated code gets legacy modes; annotations drive inference:

```ocaml
let f x = x              (* val f : 'a -> 'a *)
let g (x @ local) = x    (* val g : 'a @ local -> 'a @ local *)
```

Functions are **not mode-polymorphic**: `f` above has one mode
signature, fixed by its definition and uses in the same file.
(Inference-only mode polymorphism exists behind
`-extension mode_polymorphism_alpha` — experimental, no surface
syntax; don't rely on it. Use `ppx_template` for mode-polymorphic
code.)

### Mode Implications and Where They Apply

Some modes imply defaults on other axes:

| Written | Implies (by default) |
|---------|----------------------|
| `global` | `forkable`, `unyielding` |
| `local` | `unforkable`, `yielding` |
| `stateless` / `reading` / `writing` / `stateful` | `portable` / `shareable` / `corruptible` / `nonportable` |
| `immutable` / `read` / `write` / `read_write` | `contended` / `shared` / `corrupted` / `uncontended` |

These only affect *defaults*; every combination is expressible by
writing both axes. The implied axis is **pinned only at "fixed"
sites** — modes inside types (`val` arrows, type annotations), functor
parameters, `external`s. At "flexible" sites — parameters
`(x @ local)`, `let x @ local = …`, `(e : t @ local)`, return
annotations — the implied axis is *inferred* from use:

```ocaml
let require_portable (f : (unit -> unit) @ portable) = ()
let h (f @ stateful) = require_portable f
(* accepted: val h : (unit -> unit) @ portable -> unit *)
```

So moving an annotation between a `val` (pinned) and a `let`
parameter (inferred) can change what is accepted.

Applying a modality to a future mode takes the **meet**; applying to a
past mode takes the **join** (see `_05-modes/reference.md` in the
compiler docs).

### Curried Functions and Partial Application

The mode of a partial application is inferred from how the captured
parameter is *used*, not from its declared mode:

```ocaml
type box = { x : int }
let curry (b1 : box @ unique) (b2 : box @ unique) = ()
(* val curry : box @ unique -> (box @ unique -> unit) *)
```

The parentheses matter when reading printed types:
`a @ unique -> b -> c` means the partial application is `once`;
`a @ unique -> (b -> c)` means it is `many`. Locality is unchanged: a
function closing over a `local` parameter still can't be partially
applied to escape its region.

---

## Mode Crossing

Some types can "cross" modes - their values may be used at a stronger
mode than they have, because the axis is irrelevant for that type:

```ocaml
(* Immediates (int, char, bool, etc.) cross all modes *)
let f (x @ local) : int @ global = x  (* OK! int crosses locality *)

(* Functions don't cross linearity *)
let g (f @ once) : (int -> int) @ many = f  (* ERROR *)

(* Boxed data does NOT cross uniqueness, even if immutable *)
let h (lst @ aliased) : int list @ unique = lst
(* ERROR: This value is aliased but is expected to be unique. *)
```

- `int`, `bool`, `char`, etc. cross everything (they're immediates)
- Data **containing no functions** (even mutable, e.g. `int ref`)
  crosses portability; immutable data crosses contention — no
  annotation needed (e.g. a record of `int` and `float` fields)
- Data containing no functions crosses linearity (`int ref @ once`
  can be used as `many`; `(int -> int) list` cannot)
- Only immediates, unboxed numbers (`float#`) and function types
  cross uniqueness
- Kind annotations state crossing explicitly: `type t : value mod
  portable contended`; modalities on fields contribute
  (`{ v : t @@ shareable }` gives `value mod shareable`)

`:>` may mode-cross the *return* of a function type, on every axis
except locality:

```ocaml
let f (g : int -> int) = (g :> int -> int @ portable)
```

---

## Common Errors

- **"This value is local but is expected to be global"** — a local
  value escapes its region (e.g. is returned). Fix: `exclave_` or
  allocate globally; see
  [SKILL-STACK-ALLOCATION.md](SKILL-STACK-ALLOCATION.md).
- **"This value is aliased but is expected to be unique"**, **"…has
  already been used as unique at: …"**, **"…it is defined as once and
  has already been used"** — see
  [SKILL-UNIQUENESS.md](SKILL-UNIQUENESS.md).

---

## Effects and the Yielding Axis

A function containing `match … with effect E, k -> …` cases is forced
**nonportable and stateful**, and variables used anywhere in such a
`match` (scrutinee or cases) must be global. `Effect.perform` carries
an `unsafe_effects` alert; the replacement API threads a `local`
(hence `yielding`) handler token:

```ocaml
type _ Effect.t += E : int Effect.t
let f (h @ local) () = Effect.Safe.perform h E + 1
(* val f : Effect.Handler.t @ local -> unit -> int *)
let run () =
  Effect.Deep.Safe.try_with f ()
    { effc = fun (type a) (e : a Effect.t) ->
        match e with
        | E -> Some (fun (k : (a, _) Effect.Deep.continuation) ->
                 Effect.Deep.continue k 1)
        | _ -> None }
```

---

## Borrowing

`borrow_ e` (a prefix expression) gives a temporary `aliased` +
`local` view of a `unique` value without consuming it. Use it at call
sites where a callee reads/writes through a unique value transiently
but you don't want to give up the unique reference. It complements
`@ unique` parameters rather than replacing them.

It is valid in **exactly three positions**:

- function argument (most common): `f (borrow_ r)`
- let-binding right-hand side: `let y = borrow_ r in ...`
- match scrutinee: `match borrow_ r with _ -> ...`

Anywhere else — e.g. inside a tuple, `(borrow_ y, borrow_ y)` — is a
hard error ("The borrow_ operator must appear directly in a valid
borrowing context: - As an argument to a function application - On
the right-hand side of a let binding - As the scrutinee of a pattern
match"; a misplaced borrow often fails earlier with "This value is
local because it is borrowed").

Rules (documented in `_07-uniqueness/04-borrow.md`):

- A borrowed value is local, so the callee must accept its argument
  `@ local`.
- The original must be `many` and is unusable-as-unique within the
  implicit borrow region (the `let` body, the application, or the
  `match`).
- Only `global` values may leave a borrow region, so a stack-allocated
  (`exclave_`) result cannot escape it ("because it escapes the borrow
  region").
- **Warning 216 `use-during-borrowing`** — "This value is used while
  being borrowed."
- **Error** "This value is used as unique here, but it is being
  borrowed." — a conflict between a borrow and a unique use.
- Explicit borrows show "borrowed" in error messages rather than the
  generic "used".

```ocaml
let inspect (r : int ref @ local) = r.contents
let free (r : int ref @ unique) = ()

let example (r @ unique) =
  let n = inspect (borrow_ r) in
  free r;                (* still works - borrow didn't consume r *)
  n
```

---

See also: [SKILL-STACK-ALLOCATION.md](SKILL-STACK-ALLOCATION.md) for locality details,
[SKILL-UNIQUENESS.md](SKILL-UNIQUENESS.md) for uniqueness patterns.
