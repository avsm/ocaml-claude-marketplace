# Migrating from OxCaml 5.2.0minus-39 to 5.4.0-ox9

How to move a project from `5.2.0minus-39`
(`ocaml-variants.5.2.0+ox`, Jane Street libraries
`v0.18~preview.130.106+341`) to `5.4.0-ox9`
(`oxcaml-compiler.5.4.0-ox9`, libraries `v0.18~preview.133.03+642`).

"Before" snippets compile on minus-39, "after" snippets on ox9; error
messages are the ox9 compiler's own text. This file covers what you
have to *do*; for what changed, when and why, see
[CHANGES-39-ox9.md](CHANGES-39-ox9.md).

## How to use this guide

1. Switch the toolchain (step 0). It is all-or-nothing: compiler,
   libraries, ppx and editor tooling move together.
2. Build. Fix **errors** using the lookup table below — most are
   mechanical.
3. Triage the new **warnings and alerts** (step 9). They do not fail a
   default `dune build`; the warnings do fail `-warn-error +a` builds.
4. Check the **silent behaviour changes** (step 14) — these compile
   cleanly and differ at runtime or in tool output.
5. Re-promote expect tests last (step 13).

### Error message → fix

| ox9 says | Go to |
|----------|-------|
| `The type int64 has no unboxed version. Hint: Did you mean int64_u?` (also `int32`, `nativeint`, `float32`, `Int64.t`) | [1](#1-rename-the-unboxed-number-types) |
| `Unbound value sexp_of_int64_u` (or `compare_int64_u`, and other deriver-generated names ending in `_int64_u`, `_int32_u`, …) | [1](#1-rename-the-unboxed-number-types), deriving note |
| `Syntax error` pointing at an identifier named `effect` | [2](#2-effect-is-a-keyword) |
| `This construct requires the beta version of the extension "layouts", which is disabled and cannot be used` | [3](#3-fields-of-kind-any) |
| `All arguments of the constructor A are void, so it must be annotated with [@immediate_all_void_constructor].` | [4](#4-all-void-constructors) |
| `This local-returning application is in a tail position that is not enclosed in an exclave_ expression.` | [5](#5-local-returning-tail-calls) |
| `Functional updates that implicitly read atomic fields (here x) are forbidden.` | [6](#6-atomic-fields) |
| `Atomic record fields must have layout value.` | [6](#6-atomic-fields) |
| `Use of [%atomic.loc] with polymorphic record fields (here c) is forbidden.`; `Mutable block indices to polymorphic record fields (here c) are forbidden.` | [6](#6-atomic-fields) |
| `Integer literal exceeds the range of representable integers of type …` | [7](#7-integer-literals-one-past-the-maximum) |
| `This function's argument has type a, whose layout is known only from the GADT pattern match at …` | [8](#8-partial-gadt-matches-that-fix-a-layout) |
| `Unknown builtin primitive %caml_float_array_get128` (and other `%caml_float_array_*` vector primitives) | [10](#10-simd) |
| `Unbound value Hashtbl.choose__local`, `Unbound value D.wrap_unique`, `Unbound value …Capsule.Prim.Key.box`, `Unbound module Int64_u.Array`, `Unbound value Basement.Stdlib_shim.runtime5` | [11](#11-jane-street-libraries) |
| `migration error: effect patterns is not supported before OCaml 5.02` | [11](#11-jane-street-libraries) |
| `No implementations provided for the following modules: Env … Path …` (linking against `compiler-libs.common`) | [12](#12-build-flags-compiler-libs-c-stubs) |
| `Project should be formatted using ocamlformat version "…", but the installed version is "unknown"`; `ocamlformat: ignoring "x.ml" (syntax error)` | [0](#0-switch-the-toolchain) |
| `Program odoc not found in the tree or in PATH`; opam: `No agreement on the version of oxcaml-compiler` when installing `odoc` | [0](#0-switch-the-toolchain) |
| `sse_stubs.c: fatal error: 'simde/x86/sse.h' file not found` while installing `ocaml_simd` (arm64) | [10](#10-simd) |
| `ocamlopt: unknown option '-gdwarf-config-max-type-to-shape-depth'` / `'-flambda2-expert-cont-specialization-budget'` | [12](#12-build-flags-compiler-libs-c-stubs) |
| C: `caml_failure_exn` / `caml_raise_async_if_exception` undeclared | [12](#12-build-flags-compiler-libs-c-stubs) |

## 0. Switch the toolchain

```sh
opam update
opam switch create 5.4.0-ox9 oxcaml-compiler.5.4.0-ox9 \
  --repos ox=git+https://github.com/oxcaml/opam-repository.git,default
eval $(opam env --switch 5.4.0-ox9)
opam install dune base core ppx_jane ocamlformat merlin ocaml-lsp-server
```

- **ox9 is on the default (`main`) branch of the OxCaml opam
  repository**, so the `ox` remote used for minus-39 is the right
  one; `opam update` makes ox9 visible in it (`--repos ox=…` is
  accepted whether or not `ox` is already configured with that URL).
  `oxcaml-compiler` without a version also selects ox9 at the time of
  writing. Building the compiler takes roughly ten minutes on a fast
  machine (it bootstraps OCaml 5.4.0, dune and menhir first).
- **Libraries and compiler are locked together.** Every
  `v0.18~preview.133.03+642` package declares
  `conflicts: "oxcaml-compiler" {< "5.4.0-ox9"}`, and the
  `130.106+341` packages declare
  `conflicts: "oxcaml-compiler" {!= "5.2.0minus39"}`.
- **Nothing binary carries over.** Magic numbers went 578 → 586
  (`Caml1999I586`), so every `.cmi`/`.cmx`/`.cma`, every ppx driver
  and every tool must be rebuilt in the new switch. `dune clean` any
  workspace built with the old switch.
- **Tooling versions that match ox9**: `ppxlib 0.39.0+ox` (was
  `0.33.0+ox2`), `merlin 5.6.1-504+ox`, `ocaml-lsp-server 1.24.0+ox`,
  `ocamlformat 0.29.0+ox`, `utop 2.16.0+ox3`, upstream `dune 3.24.2`.
- **ocamlformat works, with two catches.**
  - It reports its own version as `unknown`, so a `.ocamlformat` with
    a `version = …` line is rejected (`Project should be formatted
    using ocamlformat version "0.29.0+ox", but the installed version
    is "unknown"`). Delete the `version` line or add
    `version-check = false`.
  - It formats modes, kinds, unboxed types and literals,
    comprehensions, `let mutable`, `[@atomic]`, block indices and
    `%template` code, but its parser rejects three constructs and
    skips the whole file (exit status 1, so `dune fmt` fails on it):
    `| effect E, k -> …` handler patterns, `overwrite_ e with …`, and
    `val poly_` in signatures. Keep those in files excluded from
    formatting.
- **There is no odoc for ox9.** The only odoc an OxCaml switch
  accepts is the patched `odoc.3.2.0+ox2`, which is restricted to
  minus-39 (`conflicts: "oxcaml-compiler" {… >= "5.2.0minus40"}`) and
  whose source does not compile against the ox9 `compiler-libs`.
  `opam install odoc` reports `No agreement on the version of
  oxcaml-compiler`, and `dune build @doc` fails with `Program odoc not
  found` (its `opam install odoc` hint cannot be followed). If you
  publish API docs, keep building them on the minus-39 switch until a
  5.4 odoc is released.
- **`ocaml_simd` on arm64** needs a one-line workaround to install —
  see [step 10](#10-simd).
- **Version checks.** `ocamlopt -version` prints `5.4.0+ox` for every
  ox release (it was `5.2.0+ox`); the exact release is
  `opam var oxcaml-compiler:release` → `5.4.0-ox9`. In C,
  `OCAML_VERSION` is `50400` (was `50200`). Fix any script, dune
  `enabled_if`, or `%{ocaml_version}` test that matches on `5.2`.

## 1. Rename the unboxed number types

The `#` spellings of four unboxed number types were removed (ox7).

| minus-39 | ox9 |
|----------|-----|
| `int64#` | `int64_u` |
| `int32#` | `int32_u` |
| `nativeint#` | `nativeint_u` |
| `float32#` | `float32_u` |
| `Int64.t#`, `Int32.t#`, `Nativeint.t#` | `int64_u`, `int32_u`, `nativeint_u` |

Before (minus-39 only):

```ocaml
type t = { a : int64#; b : int32#; c : nativeint#; d : float32# }
external add : int64# -> int64# -> int64# = "%int64#_add"
```

After (ox9 only — `int64_u` is unbound on minus-39):

```ocaml
type t = { a : int64_u; b : int32_u; c : nativeint_u; d : float32_u }
external add : int64_u -> int64_u -> int64_u = "%int64#_add"
```

**Do not touch** — these are unchanged:

- literals: `#42L`, `#42l`, `#42n`, `#1.0s`;
- every other `#` type: `float#`, `int#`, `int8#`, `int16#`, `char#`,
  `bool#`, `unit#`, SIMD types (`float32x4#`, `int64x2#`, …), unboxed
  tuples `#(a * b)` and unboxed records `r#`;
- primitive-name **strings**: `"%int64#_add"`,
  `"%caml_bytes_get16_indexed_by_int32#"`.

A rename that respects those rules (type positions only; skips names
preceded by `%`, `_` or `.`):

```sh
git ls-files '*.ml' '*.mli' | xargs perl -pi -e \
  's/(?<![%\w.])(int64|int32|nativeint|float32)#/$1_u/g'
git grep -nE '(Int64|Int32|Nativeint)\.t#'    # fix these by hand
```

Three follow-ups the rename does not handle:

- **Derivers (Jane Street ppx).** Base no longer exports
  `sexp_of_int64_u` & co., so a bare `int64_u` field under
  `[@@deriving sexp_of]` (or `compare`, `bin_io`, …) fails with
  `Unbound value sexp_of_int64_u`. Spell such fields with the module
  type — `Int64_u.t`, or `I64.t`, after `open Unboxed` (the `unboxed`
  library; `Base` itself has no `Int64_u`). This form also compiles
  on minus-39:

  ```ocaml
  open Base
  open Unboxed
  type t = { a : Int64_u.t; b : Float_u.t; c : Int32_u.t }
  [@@deriving sexp_of]
  ```

- **`int64_u box` is not `int64`.** The new `'a box` type constructor
  does not reduce: `(x : int64_u box)` is not accepted where `int64`
  is expected. Convert with `Int64_u.to_int64` / `of_int64`.
- **Generated or printed signatures** (`ocamlopt -i`, `.mli` files
  produced by tools, expect-test output) now print `int64_u`.

**One spelling for both compilers.** If the code must build on
minus-39 *and* ox9, use the module aliases, which exist on both:

```ocaml
module I64 = Stdlib_upstream_compatible.Int64_u
module I32 = Stdlib_upstream_compatible.Int32_u
module N = Stdlib_upstream_compatible.Nativeint_u
type t = { a : I64.t; b : I32.t; c : N.t; f : Stdlib_stable.Float32_u.t }
let succ (x : I64.t) : I64.t = I64.add x #1L
```

(With Jane Street libraries: `Int64_u.t`, `Int32_u.t`,
`Nativeint_u.t`, `Float32_u.t` after `open Unboxed`.)

## 2. `effect` is a keyword

OCaml 5.3 made `effect` a keyword, so this is now a syntax error:

```ocaml
let effect = 1
let f ~effect () = effect + 1
type r = { effect : int }
```

Rename the identifier (`eff`, `effect_`), including record fields and
labels at every use site. If you cannot rename yet, restore the old
keyword set per-library — this compiles the snippet above unchanged:

```
(library (name foo) (flags (:standard -keywords 5.2)))
```

or globally with `OCAMLPARAM='keywords=5.2,_'`. minus-39 does not
know `-keywords`, so the flag is not portable across both compilers;
the rename is.

## 3. Fields of kind `any`

Before (compiled on minus-39, which silently narrowed the annotation):

```ocaml
type ('a : any) t = { x : 'a }
type ('a : any) u = A of 'a
```

On ox9 a field or constructor argument whose type is a variable
annotated `any` (or an abstract `type t : any`) really is `any`, and
that needs `-extension layouts_beta`; `{ x : 'a list }` is
unaffected. In migrated code you almost never meant that. Drop the
annotation or state the kind you had in practice:

```ocaml
type ('a : value_or_null) t = { x : 'a }
type 'a u = A of 'a
```

Only add `-extension layouts_beta` if the type genuinely has to hold
non-value layouts.

## 4. All-void constructors

A variant constructor all of whose arguments are void (`unit#`, or an
unboxed product of voids) must be marked:

```ocaml
type t = A of unit# [@immediate_all_void_constructor] | B of int
```

The attribute is accepted (and ignored) by minus-39, so this form
builds on both.

## 5. Local-returning tail calls

minus-39 silently made a function local-returning when its tail call
returned a local value of a type that crosses locality (such as
`int`). ox9 rejects it. The usual culprits are `fst`/`snd` (and other
`[@local_opt]` primitives) applied to a local tuple as the last
expression:

```ocaml
let g (p : (int * int) @ local) = fst p
let f () = let local_ x = (1, 2) in fst x
```

Fixes, in order of preference (all four also compile on minus-39):

```ocaml
let g1 (p : (int * int) @ local) = let (a, _) = p in a   (* returns int *)
let g2 (p : (int * int) @ local) = let r = fst p in r    (* returns int *)
let g3 (p : (int * int) @ local) = fst p [@nontail]      (* returns int *)
let g4 (p : (int * int) @ local) = exclave_ fst p        (* returns int @ local *)
```

`g1`–`g3` give the function an ordinary global result. `exclave_`
keeps the minus-39 behaviour (local-returning) and pushes the
obligation onto every caller — use it only when that is what you
want. Not affected: field projection and pattern matching on a local
value, `!r`, and expressions like `fst p + snd p` (the tail call there
is `+`).

## 6. Atomic fields

**Functional update.** `{ t with … }` may no longer implicitly copy an
`[@atomic]` field:

```ocaml
type t = { mutable x : int [@atomic]; y : int; z : int }
let bump t = { t with y = 2 }
```

List every atomic field explicitly (compiles on both compilers):

```ocaml
type t = { mutable x : int [@atomic]; y : int; z : int }
let bump t = { t with x = t.x; y = 2 }
```

**Layout.** `{ mutable f : float# [@atomic] }` compiled on minus-39
and is now `Atomic record fields must have layout value.` — it was
never sound; box the field or drop `[@atomic]`. (The reverse got
easier: an atomic `value` field next to unboxed fields, an error on
minus-39, is accepted on ox9.)

**Polymorphic fields.** For a field `c : 'a. 'a option`,
`[%atomic.loc r.c]` and a mutable block index `(.c)` compiled on
minus-39 (unsound) and are now errors.

## 7. Integer literals one past the maximum

minus-39 let a literal equal to max+1 wrap to the minimum; ox9
rejects it for every integer type, boxed and unboxed:

```ocaml
let a = 4611686018427387904
let b = 9223372036854775808L
let c = #2147483648l
```

Write the negative literal or hex (compiles on both):

```ocaml
let a = -4611686018427387904
let b = -9223372036854775808L
let c = -#2147483648l
let d = 0x8000_0000_0000_0000L
```

## 8. Partial GADT matches that fix a layout

Rare. A later parameter whose layout is only known from a
*non-exhaustive* GADT match on an earlier parameter is rejected
(minus-39 gave only warning 8, and the code could segfault):

```ocaml
type ('a : any) t = V : 'a t | B : ('a : bits64) t
let f : type (a : any). a t -> a -> unit -> a = fun V x () -> x
```

Give the parameter a layout that does not depend on the match, make
the match exhaustive, or match inside the body
(`fun t -> match t with V -> fun x () -> x`):

```ocaml
type ('a : any) t = V : ('a : value) t | B : ('a : bits64) t
let f : type (a : value). a t -> a -> unit -> a = fun V x () -> x
```

## 9. New warnings and alerts

None of these fail a default build (dune's dev profile keeps them as
warnings); the warnings do fail `-warn-error +a`, the alert does not.

| Warning | Fires on | Fix |
|---------|----------|-----|
| 181 `imprecise-kind-annotation` (new, on) | `type ('a : any) w = 'a list` — annotation weaker than what is inferred | write the inferred kind: `('a : value_or_null)`; for array elements `('a : any mod separable)` |
| 183 `redundant-kind-modifier` (now on) | `type t : value maybe_separable` | delete the modifier |
| 184 `ignored-kind-modifier` (now on) | `type t : float64 non_null` | delete the modifier |
| 211 `redundant-modifier` (renamed from `mod-by-top`, widened) | `value mod global aliased` (implied), `value mod local` (default) | delete the redundant modifier |
| 220 `redundant-modality` (new, on) | `{ a : int ref @@ global aliased }`, `{ mutable x : int @@ global }`, `val z : t @@ nonportable` | write `@@ global`; no `@@ global`/`aliased`/`many` on `mutable` fields; drop default modalities |
| Alert `unsafe_effects` (new) | every `Effect.perform` | see below |

Before (clean on minus-39, six warnings on ox9):

```ocaml
type ('a : any) w = 'a list
type t1 : value maybe_separable
type t2 : float64 non_null
type t3 : value mod global aliased
type t4 : value mod local
type r = { mutable x : int @@ global }
```

After (clean on both):

```ocaml
type ('a : value_or_null) w = 'a list
type t1 : value
type t2 : float64
type t3 : value mod global
type t4 : value
type r = { mutable x : int }
```

**Stale warning names are a trap.** `-w -mod-by-top` /
`[@@@warning "-mod-by-top"]` no longer names a warning: ox9 parses the
string as single-letter warning classes, toggles unrelated warnings,
and tells you only through an `ocaml_deprecated_cli` alert. Search
build files and sources for `mod-by-top` and replace with the number
(`-211`), which works on both compilers. Likewise
`unnecessarily-partial-tuple-pattern` moved from 189 to 75 — a
numeric `-189` is silently accepted but no longer silences it; use
`-75`.

**`Effect.perform`.** Existing code compiles and runs but prints
`Alert unsafe_effects: Stdlib.Effect.perform` at each use. Either
silence it (works on both compilers):

```ocaml
[@@@alert "-unsafe_effects"]
type _ Effect.t += E : int Effect.t
let f () = Effect.perform E
```

or move to the token-passing API (`Effect.Safe.perform` with
`Effect.Deep.Safe.try_with`/`match_with`; ox9 only — see
[CHANGES-39-ox9.md](CHANGES-39-ox9.md)).

## 10. SIMD

- The vector load/store primitives on **`float array`** (and
  `float iarray`) were removed:
  `Unknown builtin primitive %caml_float_array_get128`. Switch the
  container to `floatarray` or `float# array`, whose primitives exist
  on both compilers:

  ```ocaml
  external get : floatarray -> int -> float64x2 = "%caml_floatarray_get128"
  external getu : float# array -> int -> float64x2
    = "%caml_unboxed_float_array_get128"
  ```

- Signatures that mention `int64#` etc. (`scale:`, BMI/POPCNT
  externals) need the step-1 rename. SIMD type names and primitive
  strings do not change.
- `ocaml_simd`: its API changed only by the `_u` respelling; like
  the other libraries it requires ox9. See
  [SKILL-SIMD.md](SKILL-SIMD.md).
- **arm64 only:** `opam install ocaml_simd` fails at
  `v0.18~preview.133.03+642` (`sse_stubs.c: fatal error:
  'simde/x86/sse.h' file not found` — the vendored headers are not on
  the include path). Install it with

  ```sh
  CPATH="$(opam var prefix)/.opam-switch/build/ocaml_simd.v0.18~preview.133.03+642/simde" \
    opam install ocaml_simd
  ```

  Once installed, both `ocaml_simd.sse` and `ocaml_simd.avx` work on
  arm64 (operations without a NEON builtin run as C calls).

## 11. Jane Street libraries

`v0.18~preview.130.106+341` → `v0.18~preview.133.03+642`. Breaking
changes that OxCaml-flavoured code hits:

| minus-39 libraries | ox9 libraries |
|--------------------|---------------|
| bare `int64#` field under `[@@deriving …]` | `Int64_u.t` (see step 1) |
| `Hashtbl.choose__local` | `Hashtbl.choose_local` |
| `Capsule.Prim.Data.wrap_unique ~access x` | `(Capsule.Prim.Data.wrap [@mode unique]) ~access x` |
| `…wrap_once`, `…wrap_once_unique`, `…unwrap_unique`, `access_local`, … | the base name with a mode instance: `[@mode once]`, `[@mode once unique]`, `[@mode unique]`, `(access [@mode local])`, … |
| `Capsule.Prim.Key.box k` / `Key.unbox b` | `(Capsule.Prim.Key.box [@mode.explicit global unique]) k` (no unsuffixed instance left) |
| `Capsule.Prim.Key.box_aliased` / `unbox_aliased` | `(Capsule.Prim.Key.box [@mode.explicit global aliased])` |
| Capsule `Blocking_sync` callbacks | now also require `@ unyielding` |
| `Int64_u.Array.get` / `set` / `length` (also `Int32_u.Array`, `Nativeint_u.Array`) | modules removed — `Base.Array.get` / `set` / `length` already accept an `int64_u array`; other functions via a kind instance: `(Array.map [@kind bits64 bits64])`, `(Array.init [@kind bits64])`, `(Array.copy [@kind bits64])` |
| `Int64_u.Array_index.get a i` (index is an `int64_u`; also `Iarray_index`) | `Unboxed_indexing.Array.Indexed_by_int64.get a i` — new opam package and dune library `unboxed_indexing` |
| `Basement.Primitives.box_int64` / `unbox_int64` (and `int32`, `nativeint`) | `Basement.Primitives.Int64_u.box` / `unbox` |
| `Basement.Stdlib_shim.runtime5 ()` | removed — runtime 4 is gone, so the answer is always `true`; delete the branch |

The compiler's hint names the mangled form (`Did you mean
D.wrap__unique?`); prefer the `[@mode …]` instance syntax, which
needs `ppx_template` (part of `ppx_jane`).

**Effect-handler syntax and ppx.** The new `| effect E, k -> …`
pattern fails in any file that goes through a ppx driver:
`migration error: effect patterns is not supported before OCaml 5.02`.
In preprocessed files keep using the function API
(`Effect.Deep.try_with` / `match_with` with an `effc` record), or put
the handler in a module without `(preprocess …)`.

For the non-breaking additions (`Or_error.Portable`, `*_or_null`
accessors, `Stdlib_shim.Dynamic`, …) see the library section of
[CHANGES-39-ox9.md](CHANGES-39-ox9.md) and
[SKILL-BASE.md](SKILL-BASE.md) / [SKILL-CORE.md](SKILL-CORE.md).

## 12. Build flags, compiler-libs, C stubs

**compiler-libs.** `compiler-libs.common` was split. Programs using
typing-level modules (`Path`, `Types`, `Env`, `Typemod`, …) compile
but fail to link with `No implementations provided for the following
modules`. Depend on the new library, which pulls in `common`:

```
(libraries compiler-libs.frontend)
```

`compiler-libs.frontend` does not exist on minus-39, and the
`oxcaml_utils` archive is gone. Parsing-only tools (`Parse`,
`Parsetree`, `Location`, `Longident`) still link against
`compiler-libs.common` alone. Source-level API changes
(`Compiler_hooks` parse-tree hooks now return the AST, `Clflags` and
warning-constructor renames, where the `oxcaml_utils` modules went):
"compiler-libs and tooling" in [CHANGES-39-ox9.md](CHANGES-39-ox9.md).

**Flags** (passing the old name is a hard error):

| minus-39 | ox9 |
|----------|-----|
| `-gdwarf-config-max-type-to-shape-depth` | `-type-to-shape-max-depth` |
| `-flambda2-expert-cont-specialization-budget` | `-flambda2-expert-cont-specialization-threshold` |
| building the compiler with `--disable-runtime5` | removed — runtime 4 is retired |

**C stubs.**

| minus-39 | ox9 |
|----------|-----|
| `caml_failure_exn(msg)` | `caml_exception_failure_value(msg)` |
| `caml_raise_async_if_exception(res, where)` | `caml_get_value_or_raise_async(result, where)` on a `caml_result` |
| `caml_callback_exn` + `Is_exception_result` | still available; prefer `caml_callback_res` returning `caml_result` (`caml_result_is_exception`) |
| `caml_alloc_for_heap`, `caml_free_for_heap`, `caml_add_to_heap`, `caml_stat_alloc_aligned[_noexc]` | removed |

Guard code that must build against both with
`#if OCAML_VERSION >= 50400`.

**Upstream checks that can surface new errors**: link-order checking
for `.cma`/`.cmxa` creation and native linking (5.3);
signature-avoidance tightening (5.4) — programs that relied on
silently-created abstract module types can newly fail.

## 13. Expect tests and anything that parses compiler output

Re-promote after everything else builds. What changed in the text:

- **No double quotes around identifiers** in uncoloured output:
  `This value is "local"` → `This value is local`;
  `Unbound type constructor "foo"` → `Unbound type constructor foo`.
- Many messages were reworded by the OCaml 5.3/5.4 merge (hints are
  indented differently, warning continuation lines are indented).
- Mode order in printed types changed (`local once unique`, where
  minus-39 printed `local unique once`).
- `int64#` & co. print as `int64_u`.

Scripts that scrape symbols: flat-mangled *data* symbols changed
shape (`camlFoo__add_4` → `camlFoo__add_0_1`); code symbols did not.

## 14. Silent behaviour changes — audit, nothing fails to compile

- **`List.sort_uniq`** keeps a different representative among elements
  that compare equal. With a comparison on the first component,
  `[(1,"a"); (1,"b"); (1,"c")]` gives `[(1,"b")]` on minus-39 and
  `[(1,"a")]` on ox9. Matters only when "equal" elements are
  distinguishable.
- **`Uchar.hash`** returns different values
  (`Uchar.hash (Uchar.of_int 0x1F600)`: `128512` → `410644354`). Do
  not reuse persisted hashes or `Hashtbl.Make (Uchar)` tables
  marshalled by the old compiler.
- **Inferred modes are more permissive.** A parameter closed over at
  `unique` no longer forces the partial application to `once` unless
  it is used that way, and `(f @ stateful)` at a parameter no longer
  pins `nonportable`. Code only gets *more* accepted, but inferred
  signatures (`-i`, generated `.mli`) can differ: e.g.
  `box @ unique -> box @ unique -> unit` becomes
  `box @ unique -> (box @ unique -> unit)`.
- **`-shape-format` defaults to `debugging-shapes`** (was
  `old-merlin`), changing `.cms`/`.cmt` contents for external tools.

## Not a migration problem

Things that look alarming in the changelog but cannot break code that
compiled on minus-39:

- *Effect handlers make the enclosing function nonportable and
  stateful.* That is about the new `effect` pattern syntax, which
  minus-39 did not have; `Effect.Deep.try_with` was already
  nonportable. When adopting the syntax, keep handlers out of
  `portable`/`stateless` functions.
- *`-frametables-in-rodata` removed.* The flag only existed between
  minus-40 and ox5.
- *`List.take`/`drop`, `Stdlib.Iarray`, `Dynamic`, `'a box`,
  unsigned `uint*_u` types, `idx_atomic`, `let poly_`* — all
  additions. No Stdlib, `Stdlib_stable` or
  `Stdlib_upstream_compatible` value was removed in this window.
- *Mode names, axes and `@`/`@@` syntax* are unchanged.

## Building on both compilers during the transition

| Concern | Form that works on minus-39 and ox9 |
|---------|-------------------------------------|
| Unboxed number types | `Stdlib_upstream_compatible.Int64_u.t` etc. (or `Int64_u.t` with `unboxed`) |
| All-void constructor | add `[@immediate_all_void_constructor]` |
| Tail calls | bind, destructure, `[@nontail]` or `exclave_` |
| Atomic functional update | list atomic fields explicitly |
| Literals | negative or hex spelling |
| Kind annotations | `value_or_null` / inferred kind instead of `any` |
| Warning control | numbers (`-211`, `-220`), not mnemonics |
| `Effect.perform` | `[@@@alert "-unsafe_effects"]` |
| C stubs | `#if OCAML_VERSION >= 50400` |

No common form exists for: bare `int64_u`/`int64#`, `effect` as an
identifier with `-keywords`, `Hashtbl.choose_local`, Capsule
`[@mode …]` instances, `compiler-libs.frontend`. Those need the
switch to be flipped in one commit.

## Checklist

- [ ] New `oxcaml-compiler.5.4.0-ox9` switch from the `ox` opam repository;
      libraries at `133.03+642`; `dune clean`; ppx and tools rebuilt.
- [ ] Version checks accept `5.4.0+ox` / `OCAML_VERSION 50400`.
- [ ] `int64#`/`int32#`/`nativeint#`/`float32#` → `_u`; `Int64.t#`
      forms fixed by hand; derived fields use `Int64_u.t`.
- [ ] No identifier, label or field named `effect` (or
      `-keywords 5.2`).
- [ ] `('a : any)` on fields/constructor args removed or narrowed.
- [ ] `[@immediate_all_void_constructor]` added where demanded.
- [ ] Local-returning tail calls fixed.
- [ ] Atomic fields: explicit copies in `{ t with … }`; no non-value
      atomic fields.
- [ ] max+1 integer literals rewritten.
- [ ] Warnings 181/183/184/211/220 triaged; `mod-by-top` and `-189`
      removed from build files; `unsafe_effects` handled.
- [ ] SIMD `float array` vector externals moved to `floatarray` /
      `float# array`.
- [ ] Library renames: `choose_local`, Capsule `[@mode …]`,
      `Stdlib_shim.runtime5`.
- [ ] `compiler-libs.frontend`, renamed flags, C stub names.
- [ ] `List.sort_uniq` / `Uchar.hash` uses audited.
- [ ] Expect tests re-promoted; output scrapers updated.
