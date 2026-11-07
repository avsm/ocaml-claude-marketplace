# OxCaml Changes: 5.2.0minus-39 to 5.4.0-ox9

What changed between OxCaml `5.2.0minus-39` (2026-05-22,
`ocaml-variants.5.2.0+ox`, OCaml 5.2 base) and **`5.4.0-ox9`**
(2026-09-23, `ocaml-variants.5.4.0+ox` / `oxcaml-compiler.5.4.0-ox9`,
OCaml 5.4 base): 473 merged PRs. The version naming scheme changed
along the way: `5.2.0minus-N` → `5.4.0-oxN`.

This file is the reference: what changed, when (tag, PR), and how new
features work. To port a project use
[MIGRATION-39-ox9.md](MIGRATION-39-ox9.md), the single home of upgrade
instructions.

| Tag | Date | Magic | What it is |
|-----|------|-------|------------|
| `5.2.0minus-39` | 2026-05-22 | 578 | previous target |
| `5.2.0minus-40` | 2026-06-09 | 579 (#6204) | last tag on the OCaml 5.2 base; 40 first-parent merges |
| `5.4.0-ox1` | 2026-06-12 | 580 | exactly one commit later: the upstream OCaml 5.4 merge (#5325) |
| `5.4.0-ox2` | 2026-07-02 | 581 | 98 PRs: runtime 4 retired, `'a box`, `_u` names |
| `5.4.0-ox3` | 2026-07-21 | 582 | `Dynamic`, safe effects API, warnings 181/183/184/220 |
| `5.4.0-ox4` | 2026-07-30 | 583 | `let poly_` runs, unsigned types, atomic fields in mixed blocks |
| `5.4.0-ox5` | 2026-08-14 | 584 | `compiler-libs.frontend` split, `idx_atomic`, literal range check, AVX-512 → `simd_beta` |
| `5.4.0-ox6` | 2026-08-25 | 585 | idle-phase GC, heap extension, shapes default, `-functorize` |
| `5.4.0-ox7` | 2026-09-03 | 586 | **`int64#`/`int32#`/`nativeint#`/`float32#` removed**, `addressable`, mode polymorphism (alpha) |
| `5.4.0-ox8` | 2026-09-18 | 586 | one commit: revert of a minor-GC table change |
| `5.4.0-ox9` | 2026-09-23 | 586 | one commit: arm64 relocation fix |

ox3–ox7 together are 334 merged PRs.

A tag on an item ("ox4", "minus-40" = `5.2.0minus-40`) names the first
tag containing the change. Error messages are quoted as the batch
compilers print them at ox9; unlike minus-39, **ox9 prints no double
quotes around identifiers in uncoloured error output**.

## Installing / detecting

- `ocamlopt -version` prints `5.4.0+ox` for *every* ox release;
  `Sys.ocaml_version = "5.4.0+ox"`.
- `opam var oxcaml-compiler:release` → `5.4.0-ox9` is the only exact
  check (`ocamlopt -config | grep cmi_magic` → `Caml1999I586`
  identifies ox7–ox9 as a group). In C: `#ifdef OXCAML_RUNTIME`.
- opam packages: `oxcaml-compiler.5.4.0-ox9` (note the hyphen; the old
  one was `oxcaml-compiler.5.2.0minus39`), `ocaml-variants.5.4.0+ox`.
  Both ox9 and the `133.03+642` libraries are on the default (`main`)
  branch of `github.com/oxcaml/opam-repository`; `opam update` an
  existing `ox` remote if it still stops at ox7.
- The opam build is configured `--enable-middle-end=flambda2
  --enable-runtime5 --enable-stack-checks --enable-poll-insertion
  --enable-multidomain` (same as minus-39), so `multidomain: true` and
  `poll_insertion: true` in `ocamlopt -config`.
- Matching ecosystem: Jane Street libraries
  `v0.18~preview.133.03+642` (were `preview.130.106+341`),
  `ppxlib 0.39.0+ox`, `merlin 5.6.1-504+ox`,
  `ocaml-lsp-server 1.24.0+ox`, `ocamlformat 0.29.0+ox` (reports its
  version as `unknown`; cannot parse `| effect` patterns,
  `overwrite_`, or `val poly_`). **No odoc**: the patched
  `odoc.3.2.0+ox2` is minus-39 only, so nothing installs on ox9. On
  arm64 `ocaml_simd` needs a `CPATH` workaround to build. Details and
  workarounds: MIGRATION-39-ox9.md step 0.
- dune: the libraries only require `dune >= 3.17.0`, but an OxCaml
  switch accepts only the patched `dune.3.20.2+ox`–`3.22.2+ox` or
  upstream dune ≥ 3.24.2 (the `oxcaml-dune` guard package conflicts
  with unpatched older ones); a fresh ox9 switch gets upstream
  `dune 3.24.2`.

## Summary

Breaking changes, with where each landed. Fixes are in the
[MIGRATION-39-ox9.md](MIGRATION-39-ox9.md) step given; details are in
the sections below.

| Breaking change | Landed | Step |
|-----------------|--------|------|
| Magic numbers 578 → 586 and OCaml base 5.2 → 5.4: everything, including ppx and tools, must be rebuilt; libraries are locked to the compiler | minus-40–ox7 | [0](MIGRATION-39-ox9.md#0-switch-the-toolchain) |
| `int64#`, `int32#`, `nativeint#`, `float32#` removed → `int64_u`, `int32_u`, `nativeint_u`, `float32_u` | ox7, #6663 | [1](MIGRATION-39-ox9.md#1-rename-the-unboxed-number-types) |
| `effect` is a keyword | ox1 | [2](MIGRATION-39-ox9.md#2-effect-is-a-keyword) |
| A field or constructor argument of kind `any` needs `-extension layouts_beta` (minus-39 silently narrowed the annotation) | minus-40, #5461, #6173 | [3](MIGRATION-39-ox9.md#3-fields-of-kind-any) |
| All-void constructors need `[@immediate_all_void_constructor]` | ox2, #6256 | [4](MIGRATION-39-ox9.md#4-all-void-constructors) |
| Local-returning tail calls need `exclave_` | ox7, #6396 | [5](MIGRATION-39-ox9.md#5-local-returning-tail-calls) |
| Atomic fields: no implicit copy in `{ t with … }` (ox3, #6453); must have layout `value` (minus-40, #6131, #6124); no `[%atomic.loc]` or mutable block index on a polymorphic field (ox5, #6688) | minus-40–ox5 | [6](MIGRATION-39-ox9.md#6-atomic-fields) |
| Integer literals one past the maximum are errors | ox5, #6670 | [7](MIGRATION-39-ox9.md#7-integer-literals-one-past-the-maximum) |
| A partial GADT match cannot fix a parameter's layout | ox5, #6689 | [8](MIGRATION-39-ox9.md#8-partial-gadt-matches-that-fix-a-layout) |
| Warnings 181, 183, 184, 220 on by default; 211 renamed (`mod-by-top` → `redundant-modifier`) and widened; 189 → 75; `unsafe_effects` alert on `Effect.perform` | ox1–ox5 | [9](MIGRATION-39-ox9.md#9-new-warnings-and-alerts) |
| SIMD vector primitives on `float array` / `float iarray` removed | ox2, #6128 | [10](MIGRATION-39-ox9.md#10-simd) |
| Jane Street library renames and removals | `133.03+642` | [11](MIGRATION-39-ox9.md#11-jane-street-libraries) |
| `compiler-libs.common` split, typing/lambda/driver modules now in `compiler-libs.frontend` (ox5, #6470); `oxcaml_utils` removed (ox3, #6450) | ox3–ox5 | [12](MIGRATION-39-ox9.md#12-build-flags-compiler-libs-c-stubs) |
| Flags renamed: `-flambda2-expert-cont-specialization-budget` (minus-40, #4374), `-gdwarf-config-max-type-to-shape-depth` (ox6, #5103); `--disable-runtime5` rejected, runtime 4 retired (ox2, #6218) | minus-40–ox6 | [12](MIGRATION-39-ox9.md#12-build-flags-compiler-libs-c-stubs) |
| Upstream C API removals (`caml_failure_exn`, `caml_raise_async_if_exception`, …) | ox1 | [12](MIGRATION-39-ox9.md#12-build-flags-compiler-libs-c-stubs) |
| Upstream link-order checking and signature-avoidance tightening | ox1 | [12](MIGRATION-39-ox9.md#12-build-flags-compiler-libs-c-stubs) |
| Compiler output text: messages reworded and unquoted (ox1), mode print order (ox4, #6569), flat data symbols (ox6, #6405) | ox1–ox6 | [13](MIGRATION-39-ox9.md#13-expect-tests-and-anything-that-parses-compiler-output) |
| Silent: `List.sort_uniq`, `Uchar.hash` (ox1); mode inference (ox7, #6745, #6966); `-shape-format` default (ox6, #5103) | ox1–ox7 | [14](MIGRATION-39-ox9.md#14-silent-behaviour-changes--audit-nothing-fails-to-compile) |

New features: upstream 5.3/5.4 (effect-handler syntax, `Stdlib.Iarray`,
`Pqueue`, `Pair`, `Repr`, `[| |]` literal disambiguation,
`caml_result`), `'a box`, `array#`/`iarray#`, unsigned `uint*_u`
types, `idx_atomic`, `addressable` kinds, `let poly_`/`val poly_`
(`layout_poly_alpha`), mode polymorphism (`mode_polymorphism_alpha`),
`Stdlib.Dynamic`, `Effect.Safe`, usable preemption,
`caml_region_begin/end`, `caml_add_blocks_to_heap`, `ocamlfilt`,
`-open-cmi`, `-functorize`, `-keywords`.

## Upstream OCaml 5.4 merge (ox1, #5325)

`5.4.0-ox1` is `5.2.0minus-40` plus one commit ("Merge upstream 5.4"):
the base moves from OCaml 5.2 to 5.4.0, bringing all upstream 5.3 and
5.4 features. Upstream C API changes are under [C API](#c-api).

### Language

- **Effect-handler syntax** (5.3): deep handlers directly in `match`
  and `try`.

  ```ocaml
  type _ Effect.t += Poke : unit Effect.t

  let f c i =
    match c i with
    | v -> v
    | exception Not_found -> 0
    | effect Poke, k -> Effect.Deep.continue k ()
  ```

  OxCaml caveat (ox2, #6230): a function containing effect cases is
  forced **nonportable and stateful**, so effect cases are not allowed
  in `portable`/`stateless` functions: `Error: The pattern match with
  effect cases is nonportable but is expected to be portable ...`.
  The pattern is also rejected in any file preprocessed by a ppx
  (MIGRATION step 11).
- **`effect` is a keyword** (5.3): by default all keywords are
  enabled, so `effect` as an identifier no longer parses. OxCaml
  keyword editions give three escape hatches:
  - `-keywords 5.2` (per compilation; also accepted by `ocamldep`;
    `-keywords 5.2+effect` adds it back);
  - `OCAMLPARAM='keywords=5.2,_'` (the `,_` separator is required;
    without it the variable is rejected with warning 46 and ignored);
  - OxCaml-only: bake the default into the compiler with
    `./configure --enable-keyword-edition=5.2` (supports e.g.
    `5.2+effect`).

  OxCaml's own keywords (`borrow_`, `exclave_`, `kind_`, `local_`,
  `poly_`, `stack_`, …) are unconditional regardless of edition.
- **Atomic record fields** (5.4; OxCaml already had a variant): for
  `type t = { mutable readers : int [@atomic] }`,
  `[%atomic.loc t.readers] : int atomic_loc`; `Stdlib.Atomic.Loc`
  provides operations on `'a atomic_loc`, with OxCaml mode/jkind
  annotations layered on. OxCaml restrictions: see
  [Atomic fields](#atomic-fields).
- **`'a iarray` and `Stdlib.Iarray`** are now upstream (5.4).
  `Stdlib_stable.Iarray` remains as a compat shim, but its interface
  is **narrower**: it lacks `equal`, `compare`, `find_index`,
  `find_mapi` (so does `IarrayLabels`). Bare `Iarray.` now resolves to
  `Stdlib.Iarray`, the richest plain API; `open Stdlib_stable` shadows
  it with the narrower module. `Stdlib_stable.IarrayLabels` is
  retained (upstream has no labels version) and is still the only home
  of `( .:() )` and the `*_local` variants.
- **Array literals disambiguate by expected type** (5.4):
  `[| e1; e2 |]` can mean `array`, `iarray`, or `floatarray` depending
  on context (default `array`; non-principality warning under
  `-principal`). `[: ... :]` remains an OxCaml extension (enabled by
  default).
- **UTF-8 source files** and Latin-9-compatible Unicode identifiers.
- Not new: short dependent functor types `(X : A) -> B` were already
  accepted at minus-39 (upstream 5.1); labeled tuples are upstream in
  5.4, but OxCaml had reconciled to the upstream version before
  minus-39 (#5692).
- **Error messages reworded** throughout (5.3's "The constant 42 has
  type..." style, realigned hints), and identifiers in uncoloured
  error output are no longer double-quoted: minus-39 printed
  `has type "string"`, ox9 prints `has type string` (the compiler's
  own expect-test harness still shows quotes). Expect-test output
  changes wholesale; scripts that grep compiler stderr for `"name"`
  break.
- New warning for `t as 'a` with unused `'a`
  (`unused-type-declaration`). Warnings 74 and 75: see
  [Warnings](#warnings).
- **Link-order checking** for `.cma`/`.cmxa` creation and native
  linking (5.3) — can surface new errors in existing builds.
- **Signature-avoidance tightening** (5.4): programs that relied on
  silently-created abstract module types can newly fail.

### Stdlib

New modules: **`Iarray`**, **`Pqueue`** (priority queues; exposes
`MakeMin`, `MakeMax`, `MakeMinPoly`, `MakeMaxPoly`), **`Pair`**,
**`Repr`** (`Repr.phys_equal`/`Repr.compare` as explicit replacements
for `==`/polymorphic compare).

Additions to existing modules: `Char.Ascii`; `Gc.ramp_up`,
`Gc.ramp_down`; `String.edit_distance` / `String.spellcheck`;
`Result.Syntax` (`let*`), `Result.product`, `Result.error_to_failure`,
`Result.retract`, `Result.get_ok'`; `Dynarray` gains
`unsafe_to_iarray`, `equal`, `compare`, `mem`, `find_index`, `blit`,
`exists2`, `for_all2`, `find_opt`, `find_map`, etc. (and moved to an
unboxed representation in 5.3); `List.take`/`drop`/`take_while`/
`drop_while`/`singleton`; `Seq.filteri`, `Seq.singleton`;
`Either.get_left`/`get_right`, `Either.retract`;
`Bool.logand`/`logor`/`logxor`; `Queue.drop`; `Sys.signal_to_string`,
`Sys.signal_of_int`/`to_int`, `Sys.sigio`, `Sys.sigwinch` (and
`type Sys.signal = int`); `Unix.sigwait`; `Array.equal`/`compare`,
`Float.Array.equal`/`compare`; `Format.format_text`;
`Uchar.seeded_hash`, `Uchar.utf_8_decode_length_of_byte`;
`Runtime_events.Timestamp.get_current`. (`Dynarray`, `Atomic.Loc`,
`Sys.poll_actions`, `Domain.self_index` and
`Thread.set_current_thread_name` were already present at minus-39.)

Behaviour:

- `List.take`/`drop` return `[]`/the whole list for negative n (only
  upstream 5.3 raised).
- `List.sort_uniq` keeps the *first* occurrence of duplicates.
- `Uchar.hash` is no longer `Uchar.to_int` — persisted
  `Hashtbl.Make(Uchar)` tables need rebuilding.
- `Filename.get_temp_dir_name` uses system calls on Windows.

OxCaml re-oxidised the merged stdlib: signatures carry modes and jkinds
(e.g. `Iarray`: `type (+'a : any mod separable) t`, `[@@layout_poly]`;
`List.take : ('a : value_or_null). ...`; `@@ portable` module
headers). Upstream-conformant code works unchanged.

### Where OxCaml deliberately diverges

- **Multiple domains are still gated by `--enable-multidomain`** (off
  for a bare `./configure`, **on in the opam package**, as at
  minus-39); `Domain.spawn` still carries the `do_not_spawn_domains`
  alert (pointing at `Multicore`: `Multicore.spawn`,
  `spawn_on ~domain`, unchanged) and the `unsafe_multidomain` alert
  ("Use [Domain.Safe.spawn]").
- `Stdlib_stable` compat layer kept (plus OxCaml-only modules:
  `Float32`, `Int8`/`Int16`, `Or_null`, `Idx_imm`/`Idx_mut`, …).

## Unboxed types, kinds and layouts

### `int64#`, `int32#`, `nativeint#`, `float32#` removed (ox7, #6663)

The biggest source change in the window. `int64_u`, `int32_u`,
`nativeint_u` (ox2, #6321) and `float32_u` (ox3, #6459) were
introduced as aliases of the `#` names; at ox7 the `#` names were
deleted (they aren't true unboxed versions of the custom-block types)
and the `_u` names became standalone predefined types. `int64#` now
gives `The type int64 has no unboxed version. Hint: Did you mean
int64_u?`. Rename recipe and spellings that build on both compilers:
MIGRATION step 1.

- Alias forms break too, with no hint: `Int64.t#`, `Int32.t#`,
  `Nativeint.t#` → `The type Int64.t has no unboxed version.`
- Unchanged: literals (they have the `_u` types); every other `#`
  type — `float#`, `int#`, `int8#`, `int16#`, `char#`, `bool#`,
  `unit#`, the SIMD `int8x16#` family, `Float.t#`, `r#` for records
  (there is no `float_u`/`int8_u`/`int_u` type); `%`-primitive names
  — `"%int64#_add"`, `"%array_safe_get_indexed_by_int32#"`,
  `"%box_int64"` (`"%int64_u_add"` is rejected); kinds (`bits32`,
  `bits64`, `word`, `float32`).
- **Library modules keep their names**; only `type t` changed:
  `Stdlib_upstream_compatible.{Int32_u,Int64_u,Nativeint_u}.t`,
  `Stdlib_stable.Float32_u.t`; the signatures of
  `Stdlib_stable.Idx_imm`/`Idx_mut`/`Float32` are re-spelled the
  same way.
- **`int64` and `int64_u box` are distinct types** (likewise `int32`,
  `nativeint`, `float32`): boxed `int64` is a custom block, not the
  boxing of `int64_u`. There is no `%box` primitive yet, so
  `int64_u box` cannot be constructed; `Int64_u.to_int64`/`of_int64`
  still convert to the ordinary boxed type.

### `'a box` (ox2, #5342)

A predefined covariant type constructor (parameter kind `any`, result
kind `value`) denoting the boxed version of a type; it reduces during
unification. No extension flag or new syntax.

- `float# box = float`, `int# box = int`, `int8# box = int8`,
  `bool# box = bool`, `'a ref# box = 'a ref`,
  `int8x16# box = int8x16`, `#(int * string) box = int * string`; for
  `type r = { x : int }`, `r# box = r`; for
  `type ('a : any) b = 'a box`, `float# b = float`.
- At ox2–ox6 `int64# box = int64` etc. also held; from ox7
  `int64_u box`, `int32_u box`, `nativeint_u box` and `float32_u box`
  do **not** reduce and are distinct from the boxed number types.
- Nominality is preserved: for `type u = #{ x : int }`, `u box` is
  *not* compatible with a distinct record type of the same shape.
  Float records (which have no unboxed version) don't unify with
  `_ box`. Unreduced applications print as e.g. `int box`.

### Fields and constructor arguments of kind `any` (minus-40, #5461, #6173)

Record fields, constructor arguments, inline-record fields, and
unboxed-record fields may have kind `any`, gated by
`-extension layouts_beta`.

```ocaml
type ('a : any) t = { fst : 'a; mutable snd : 'a }   (* accepted (beta) *)
let a : float# t = { fst = #1.0; snd = #2.0 }
let fst (t : int t) = t.fst      (* ok: instantiated at a representable type *)
let fst (type a : any) (t : a t) = t.fst
(* Error: Record element types must have a representable layout. *)
```

- Breaking: minus-39 accepted the same declaration *without* any
  extension, silently narrowing `('a : any)` to `value_or_null`.
  Without `layouts_beta` it is now `Error: This construct requires the
  beta version of the extension "layouts", which is disabled and
  cannot be used` (MIGRATION step 3).
- The gate fires whenever a kind-`any` variable (or abstract
  `type t : any`) is used *directly* as a field/argument type;
  `{ x : 'a list }` is unaffected.
- Sort inference flows through `any` fields since ox4/ox6 (#6436,
  #6496): `let f x = A x` can be used at `float#`. `any`-field
  representations are finalised at translation (#6496 ox6, #6436 ox4,
  #6403 ox4).
- Follow-up fixes (minus-40): sort variables for `any` fields (#6145)
  and constructor patterns (#6156), representation refinements
  (#6143), topological jkind computation for recursive groups (#6177,
  #6209).

### `[@@flatten_floats]` becomes optional (minus-40, #6106)

At minus-39 a mixed `float`+`float#` record such as
`type t = { f : float; u : float# }` *requires* `[@@flatten_floats]`
(error without it). From minus-40 it typechecks without the attribute
and the default representation is **non-flat** (the boxed `float`
field is stored as a pointer); `[@@flatten_floats]` opts back into
flat storage. Non-flat mixed records gain unboxed versions (`t#`
works); `[@@flatten_floats]` records still do not.

### All-void constructors need an attribute (ox2, #6256)

A constructor whose arguments are all void (or products of voids) must
be annotated, e.g.
`B of unit# * #(unit# * unit#) [@immediate_all_void_constructor]`;
nullary and non-void constructors are unaffected. Without it: `Error:
All arguments of the constructor A are void, so it must be annotated
with [@immediate_all_void_constructor].` The annotation opts into a
planned representation change (tagged immediate → empty block).

### Custom `[@@or_null]` types (ox2, #5830; ox4, #6500; ox5, #6383)

Still exactly two constructors (one nullary, one unary), but from ox2
the payload can be anything:

```ocaml
type no_param = A | B of int [@@or_null]
type ('a, 'b) multi = Nope | Yep of ('a list * 'b) [@@or_null]
type fn = No_fn | Fn of (unit -> unit) @@ portable [@@or_null]
type f = No_float | Fl of float [@@or_null]
    (* result kind loses non_float/separable *)
```

- Still rejected: >2 constructors, multi-argument payloads
  (`B of int * int`), GADT constructors.
- A custom `[@@or_null]` type has the same precise kind as the builtin
  (`value_or_null mod everything with <payload>`), so `int maybe`
  crosses contention etc. (ox4, #6500).
- The null constructor may carry a **void payload**, with no
  `[@immediate_all_void_constructor]` (ox5, #6383):

  ```ocaml
  type vn = Nil of unit# | Cons of string [@@or_null]
  let show = function Nil #() -> "nil" | Cons s -> s
  ```

### Unsigned unboxed integers (#4936, ox3; #6534, #6634, ox4)

- New predefined opaque types `uint8_u : bits8`, `uint16_u : bits16`,
  `uint32_u : bits32`, `uint64_u : bits64`, `unativeint_u : word`
  (under the default-on `small_numbers` extension). No literals, no
  operations, no stdlib module; distinct from the signed types.
  Convert with `external to_u8 : int8# -> uint8_u = "%identity"`.
- New unsigned division/remainder primitives for every integer width,
  boxed and unboxed: `%int_unsigned_div`, `%int64_unsigned_div`,
  `%int8#_unsigned_div`, `%int64#_unsigned_div`, … and `_unsigned_mod`.
  `Stdlib_stable.{Int8,Int8_u,Int16,Int16_u}.unsigned_div/unsigned_rem`
  now use them.

  ```ocaml
  external udiv64 : int64_u -> int64_u -> int64_u = "%int64#_unsigned_div"
  (* udiv64 (-#1L) #2L = #9223372036854775807L *)
  ```

### Integer literals no longer wrap at max+1 (ox5, #6670)

A decimal literal equal to `max_int + 1` of its type used to wrap to
the minimum. It is now an error for every width, boxed and unboxed:
`128s`, `32768S`, `#2147483648l`, `9223372036854775808L`,
`#4611686018427387904m`, `4611686018427387904` (`Error: Integer
literal exceeds the range of representable integers of type int8`).
Negative minima (`-128s`, `-#9223372036854775808L`) and hex literals
filling the width (`0xFFs`, `#0xFFFFFFFFl`) are still accepted.

### Layout polymorphism: `let poly_` / `val poly_`

**Requires `-extension layout_poly_alpha`.** Full rule list:
SKILL-KINDS.md.

- **`val poly_` in signatures** (ox2, #6287): layout-polymorphic value
  descriptions.

  ```ocaml
  val poly_ mk : 'a -> 'b -> #('a * 'b)
  val poly_ baz : ('a : immediate). 'a -> 'b -> #('a * 'b)
  (* only FREE variables are layout-polymorphic: here just 'b *)
  ```

  New **warning 219** (no mnemonic) when there are no
  layout-polymorphic variables — e.g. if every variable is explicitly
  quantified. Combining `poly_` with `layout_` quantifiers is an
  error.
- **Empty `let poly_` is an error** (ox2, #6195), not warning 217
  (which was removed): `Error: This binding has no layout variables,
  so poly_ has no effect.`
- **`let poly_` runs** (#5870, #6446 ox4; #6049 ox5), native and
  bytecode; at ox2 it only type-checked.

  ```ocaml
  (* ocamlopt -extension layout_poly_alpha *)
  let poly_ id x = x
  let poly_ pair x y = #(x, y)
  let #(a, b) = pair "a" #1L       (* string, int64_u *)
  let c = id #42.0                 (* float# *)
  ```

  `-i` and error messages print `val poly_ id : 'a -> 'a` (#6446). The
  right-hand side must be a syntactic value; `let rec poly_` is not
  polymorphic (warning 218); a `poly_` value does not match an
  ordinary `val` in a signature; cross-module use needs the defining
  `.mli` to start with `@@ static` **and** the client to be compiled
  with `-Ix <dir>` for the directory holding its `.cmx` (plain `-I`,
  or the module sitting in the current directory, gives "The value
  A.id is dynamic but is expected to be static").

### Kind annotations

- **`[@@@implicit_kind]` in structures** (ox2, #6114): the floating
  attribute, previously signature-only, now works in structures and at
  the module toplevel. Implicit kinds are lexical defaults, not part
  of the interface; they don't survive `include`.

  ```ocaml
  module M = struct
    [@@@implicit_kind: ('elt : bits64)]
    let f : 'elt -> 'elt array = fun x -> [| x |]
  end
  ```

  An implicit `('a : any)` works for GADTs instantiating `'a` at
  different layouts (ox3, #6429).
- **Scannable axes on abstract kinds** (ox2, #5874):

  ```ocaml
  module type S = sig
    kind_ k
    val get : ('a : k separable). 'a array -> 'a
  end
  ```

- **Scannable axes on arbitrary kinds** (ox2, #6367): parenthesized
  kinds, products, `mod`-kinds, `with`-kinds all accept a trailing
  axis: `type t : (value & value) non_pointer` (with warning 184
  `ignored-kind-modifier` where the axis has no effect). Parsetree
  jkind-annotation shape changed — relevant to ppx authors.
- **`addressable`** (ox7, #6712): new postfix kind modifier — a kind
  whose boxed form can have a pointer taken into it. `value`,
  `bits64`, `word`, the vector kinds and their products already are;
  `bits8/16/32`, `float32`, `float64`, `void` are not. Not used by the
  stdlib yet; no runtime-representation change.

  ```ocaml
  type t8 : bits8 addressable
  kind_ k2 = bits8 addressable
  type ('a : any) ptr                 (* your own type *)
  val ptr_of : ('a : any addressable). 'a box -> 'a ptr
  ```

### `array#` and `iarray#` (ox3, #6361)

`'a array#` / `'a iarray#` are the unboxed versions of the array
types (kind `any`). Type-level only: `'a array# box = 'a array`; you
cannot yet pass or bind a value of type `'a array#`.

### Raw-pointer primitives (ox2, #6350, #6295)

New compiler primitives for off-heap access (no stdlib module — declare
the `external`s yourself):

```ocaml
external get_ext_ptr : ('a : any). int64_u @ local -> 'a @ local
  = "%unsafe_get_ext_ptr" [@@layout_poly]
external set_ext_ptr : ('a : any). int64_u @ local -> 'a @ local -> unit
  = "%unsafe_set_ext_ptr" [@@layout_poly]
(* also %unsafe_get_ext_ptr_imm *)
```

These take a raw address in an `int64_u` (base+offset with NULL base) —
a supported replacement for `Obj.magic` pointer tricks. On bytecode
they compile but raise `Failure` at runtime. Writes to scannable
fields go through `caml_modify` on native. Separately, the
pre-existing interior-pointer primitives `%unsafe_get_ptr` /
`%unsafe_set_ptr` / `%unsafe_get_ptr_imm` (taking `#(base, offset)`)
are now implemented on bytecode too (#6295).

### Partial GADT matches cannot fix a parameter's layout (ox5, #6689)

Rare; fixes real segfaults. A parameter whose layout is known only
from a non-exhaustive GADT match on an earlier parameter is rejected;
minus-39 accepted it with only warning 8 (example and fix: MIGRATION
step 8).

```
Error: This function's argument has type a,
       whose layout is known only from the GADT pattern match at …
       That match is not exhaustive, so a caller could reach this argument at a different layout.
       Function arguments and results must be representable independently of any non-exhaustive match.
```

Related miscompile fixes with no source change: #6533 (ox3), #6356
(ox4).

### Smaller items

- **`let rec` through non-value fields of mixed blocks** (ox6, #6847):
  `type r = { a : #(r option * float#) }
   let rec r = { a = #(Some r, #4.0) }` is accepted.
- **Bytecode deep-copies unboxed-product elements** on array/idx
  reads and writes (ox2, #6123), matching native semantics (previously
  mutation through one "copy" was observable through another on
  bytecode only).
- Soundness/panic fixes (minus-40): `or_null` array segfault (#6185);
  array types with bottom-kind elements rejected (#6134).
- Soundness/miscompile fixes with no source change (ox3–ox9): `char#`
  literal matching for chars ≥ 128 (#6535), matches on polymorphic
  variants carrying `or_null` (#6996), `int64 → float32`
  constant-folding double rounding (#6474), `with`-bounded kinds in
  `module rec` (#6395), spurious "has no unboxed version" under
  `-short-paths` (#6458), element layouts for arrays of products
  (#6387), fewer `caml_modify` calls for immediate existentials in
  mixed blocks (#6631, #6633).

## Atomic fields

| Behaviour | PR | Tag | minus-39 | ox9 |
|-----------|----|-----|----------|-----|
| Atomic field of non-`value` layout, e.g. `{ mutable f : float# [@atomic] }` | #6131, #6124 | minus-40 | some cases accepted (unsound) | **error** |
| Atomic `value` field in a mixed record: define, read, write | #6423, #6545 | ox3 / ox4 | error | **accepted** |
| `{ t with y = 2 }` where another field `x` is atomic | #6453 | ox3 | accepted | **error** |
| `[%atomic.loc r.c]` where `c : 'a. 'a option` | #6688 | ox5 | accepted (unsound) | **error** |
| mutable index `(.c)` to a polymorphic field | #6688 | ox5 | accepted (unsound) | **error** |
| `[%atomic.loc t.x]` on a mixed record | #6545 | ox4 | n/a | error |
| block index `(.x)` on an atomic field | #6667 | ox5 | error | `idx_atomic` |

- Layout (#6131, #6124): `Error: Atomic record fields must have layout
  value.` — a soundness fix (some non-`value` cases previously slipped
  through single-field and inline records). A record with atomic
  fields gets no unboxed version: `type u = t_atomic#` gives `Error:
  The type t_atomic has no unboxed version. Hint: Records with
  [@atomic] fields don't get unboxed versions.`
- Mixed blocks: at minus-39 and minus-40 the error was "Atomic record
  fields are not permitted in mixed blocks".
- Functional update:

  ```
  Error: Functional updates that implicitly read atomic fields (here x)
         are forbidden. Hint: if you intend to copy the value
         of an atomic field, do so explicitly: { t with x = t.x }
  ```

**Atomic block indices** (#6667 ox5, #6743 ox7): new predefined type
`('a, 'b) idx_atomic : bits64` and module `Stdlib_stable.Idx_atomic`:

```ocaml
open Stdlib_stable
type t = { mutable x : int [@atomic]; y : string }
let i : (t, int) idx_atomic = (.x)
let () =
  let r = { x = 1; y = "a" } in
  Idx_atomic.set r i 5;
  Idx_atomic.incr r i;
  let old = Idx_atomic.fetch_and_add r i 10 in       (* 6 *)
  assert (Idx_atomic.compare_and_set r i 16 100 && old = 6)
```

`get`/`set` exist from ox5; `exchange`, `compare_and_set`,
`compare_exchange`, `fetch_and_add`, `add`, `sub`, `logand`, `logor`,
`logxor`, `incr`, `decr` from ox7. Works on atomic fields of mixed
records and through unboxed records (`(.x.#y)`); no atomic array
indices. Matching C entry points in `caml/memory.h`:
`caml_atomic_load_field`, `caml_atomic_set_field`,
`caml_atomic_exchange_field`, `caml_atomic_compare_exchange_field`,
`caml_atomic_fetch_add_field`, `caml_atomic_{add,sub,land,lor,lxor}_field`.

## Modes, stack allocation, uniqueness

No mode axis, mode name, legacy keyword or `@`/`@@` syntax was added
or removed in the whole window. What changed is inference, warnings
(220, see [Warnings](#warnings)) and one tail-call rule.

### Local-returning tail calls must be under `exclave_` (ox7, #6396)

If an application in tail position returns `local`, minus-39 silently
made the enclosing function local-returning whenever the result type
mode-crossed locality (e.g. `int`) — leaking the caller's region. ox9
rejects it: `Error: This local-returning application is in a tail
position that is not enclosed in an exclave_ expression.` The trap is
`[@local_opt]` primitives such as `fst`/`snd` on a local tuple;
`Dynamic.with_temporarily` is local-returning, so it hits this rule in
tail position too. Fixes and unaffected forms: MIGRATION step 5.

### Mode implications are inferred at annotation sites (ox7, #6745)

The implication table is unchanged (`local` ⇒ `yielding`,
`unforkable`; `global` ⇒ `unyielding`, `forkable`; `stateless` /
`reading` / `writing` / `stateful` ⇒ `portable` / `shareable` /
`corruptible` / `nonportable`; `immutable` / `read` / `write` /
`read_write` ⇒ `contended` / `shared` / `corrupted` /
`uncontended`). What changed is where it *pins* the implied axis:

- **Fixed sites** (pinned as before): modes inside types (`val`
  arrows, type annotations), functor argument/return modes,
  `external`s.
- **Flexible sites** (now only a default; the implied axis is
  inferred): parameters `(x @ local)`, `let x @ local = …`,
  expression constraints `(e : ty @ local)`, return annotations.

```ocaml
let require_portable (f : (unit -> unit) @ portable) = ()
let h (f @ stateful) = require_portable f
(* minus-39: Error: This value is nonportable but is expected to be portable.
   ox9:      val h : (unit -> unit) @ portable -> unit *)
```

Accepts more programs; inferred signatures can differ (e.g.
`'a @ local unyielding -> unit`).

### Curry modes follow use, not declaration (ox7, #6966)

Closing over a `unique`/`uncontended`/… parameter no longer forces
the partial application to `once`/`nonportable`/… unless the
parameter is actually *used* that way:

```ocaml
type box = { x : int }
let curry (b1 : box @ unique) (b2 : box @ unique) = ()
(* minus-39: val curry : box @ unique -> box @ unique -> unit
   ox9:      val curry : box @ unique -> (box @ unique -> unit) *)
```

The parentheses are significant: `a @ unique -> b -> c` means the
partial application is `once`; `a @ unique -> (b -> c)` says it is
`many`. Accepts more programs; when it does fail the error now points
at the parameter ("The pattern is contended because it is used inside
the function at … which is expected to be portable").

### Smaller mode changes

- **Coercions can mode-cross on function returns** (ox3, #6407), on
  every axis except locality:
  `(g :> int -> int @ portable)` for `g : int -> int`.
- **Middle-element modalities mode-cross** (ox7, #6272):
  `type s : value mod shareable = { v : t @@ shareable } [@@unboxed]`
  is accepted (also `shared`, `corrupted`, `corruptible`, `read`,
  `write`, `reading`, `writing`).
- **GADT constructor-local bounds are kept** (ox6, #6202):
  `type 'a t : value mod portable contended =
   | K : ('b : value mod portable contended). 'b -> 'b t`.
- **zero_alloc: an `external` may implement a `val … [@@zero_alloc]`**
  (ox5, #6651). `[@@noalloc]` C stubs are trusted; `%`-primitives are
  really checked (`"%int32_add"` on boxed `int32` fails with
  "allocation of 24 bytes for boxed_int32").

  ```ocaml
  module M : sig
    val to_int : float -> int [@@zero_alloc]
  end = struct
    external to_int : float -> int = "caml_int_of_float" [@@noalloc]
  end
  ```

- **Printing** (ox4, #6569): mode order in printed types changed
  (axes now print in a fixed order — locality, forkable, yielding,
  linearity, uniqueness, statefulness, visibility, portability,
  contention, staticity: `local once unique`, not `local unique
  once`), and `static` is printed. Affects `-i` output and expect
  tests.
- **Reworded errors** (ox7, #6249, #7027): "The parameter is local but
  is expected to be global." / "The function return is local but is
  expected to be global." replace the old "This function takes a
  parameter which is …" forms. `stack_ 42` now says "Stack allocating
  literals is not supported; they are not allocated at runtime."
  (ox3, #6471).
- **Mode polymorphism** (ox7; #5478, #5483, #6279, #6966):
  inference-only, **no surface syntax**, behind
  `-extension mode_polymorphism_alpha` (`mode_polymorphism` and
  `_beta` are accepted no-ops). With it, `let id x = x` generalises
  over modes; `-extension mode_polymorphism_printing` shows
  `val id : 'a @ [< 'm] -> 'a @ [> 'm]`. Experimental — don't rely on
  it.
- **Staticity** (`static` < `dynamic`; #6226, #6630, #6194): functors
  and `-Ix` directories are tracked; only matters for layout
  polymorphism and metaprogramming.
- **Borrowing**: no behaviour change. The uniqueness docs were
  renumbered (`_07-uniqueness/{01-intro,02-reference,03-pitfalls,
  04-borrow}.md`) and document that stack-allocated (`exclave_`)
  values cannot escape a `borrow_` region ("because it escapes the
  borrow region").

## Warnings

Default warning set went from
`+a-4-7-9-27-29-30-32..42-44-45-48-50-60-66..70-183..184` to
`+a-4-7-9-27-29-30-32..42-44-45-48-50-60-66..70-74-221`.

| No. | Name | Change | Default | PR / tag | Meaning |
|-----|------|--------|---------|----------|---------|
| 74 | `degraded-to-partial-match` | added | off | upstream / ox1 | match compiled as partial though it looks total |
| 75 | `unnecessarily-partial-tuple-pattern` | **renumbered from 189** | on | upstream / ox1 | tuple pattern ends in `..` needlessly |
| 181 | `imprecise-kind-annotation` | added | **on** | #5362 / ox3 | kind annotation weaker than the inferred kind |
| 183 | `redundant-kind-modifier` | off → **on** | on | #6360 / ox3 | scannable-axis modifier already implied |
| 184 | `ignored-kind-modifier` | off → **on** | on | #6360 / ox3 | scannable-axis modifier has no effect on this layout |
| 211 | `redundant-modifier` | **renamed from `mod-by-top`**, widened | on | #5436 / ox5 | `mod` bound is the axis default or implied by another |
| 213 | (unnamed) | **removed** | — | #5532 / ox3 | subsumed by 220 |
| 217 | (unnamed) | **removed** | — | #6195 / ox2 | empty `let poly_` is now an error |
| 219 | (no mnemonic) | added | on | #6287 / ox2 | `val poly_` with no layout-polymorphic variables |
| 220 | `redundant-modality` | added | **on** | #5532 / ox3 | modality repeated/overridden on one axis |
| 221 | `unused-alert-disable` | added | off | #6524 / ox4 | `[@alert "-foo"]` that suppressed nothing |

Triggers (ox9 output):

```ocaml
type ('a : any) w = 'a list
(* Warning 181 [imprecise-kind-annotation]: The type variable `'a'
   was annotated with kind `any'
   but was inferred to have kind `value_or_null'. *)

type t1 : value maybe_separable
(* Warning 183 [redundant-kind-modifier]: This kind modifier, or a
   stronger one, is already implied by the kind value. *)

type t2 : float64 non_null
(* Warning 184 [ignored-kind-modifier]: The kind modifier(s) non_null
   have no effect on the kind float64. *)

type t3 : value mod global aliased
(* Warning 211 [redundant-modifier]: This modifier is redundant
   because it is implied by global. *)

type t4 : value mod local
(* Warning 211 [redundant-modifier]: This modifier is redundant
   because local is the default bound on its axis, so it has no effect. *)

type r = { mutable x : int @@ global }
(* Warning 220 [redundant-modality]: This modality is redundant. *)
```

- These only break builds using `-warn-error +a`/`@a` (dune's dev
  profile keeps them as warnings). Fixes: MIGRATION step 9.
- 181 hits array code often: `('a : any) … 'a array` is inferred as
  `any separable`, so write `('a : any mod separable)`.
- **220** fires on a modality that does not change the resolved
  modality: implied by another written one, equal to the default for
  that position (including the implicit modalities of a `mutable`
  field, which are `global many aliased …`), or repeated on one axis.
  So write `@@ global`, not `@@ global aliased`; don't put
  `@@ aliased` or `@@ many` on `mutable` fields. `@@ global unique` is
  still a hard error. Silence with `-w -220`.

  ```ocaml
  type t = { a : int ref @@ global aliased }     (* aliased: implied by global *)
  type u = { mutable n : int ref @@ aliased }    (* implied by mutable *)
  type x = { global_ y : string @@ global }
  module type S = sig
    val y : int ref @@ stateless portable        (* portable: implied by stateless *)
    val z : int ref @@ nonportable               (* the default *)
  end
  (* Warning 220 [redundant-modality]: This modality is redundant. *)
  ```

- **Stale mnemonics are dangerous**: `-w -mod-by-top` and `-w -189`
  are silently accepted but no longer do what they did (MIGRATION
  step 9).
- Alert `unsafe_effects` on `Effect.perform`: next section.

## Effects, preemption, `Dynamic` and other stdlib changes

### `Effect.perform` alert and the "safe" effects API (ox3, #6281)

`Effect.perform` now carries an alert:

```ocaml
external perform : 'a t -> 'a = "%perform"
[@@alert unsafe_effects "Use [Effect.Safe.perform]. [Effect.perform] may not \
                         function correctly on Js_of_ocaml"]
```

Existing code still compiles and runs but each use prints `Alert
unsafe_effects` (not fatal under `-warn-error +a` or dune's dev
profile). Silence with `[@@@alert "-unsafe_effects"]`,
`(Effect.perform [@alert "-unsafe_effects"])`, or
`-alert -unsafe_effects`.

The new API threads a zero-size *handler token* (`Effect.Handler.t`,
layout `void`) from the handler installation to each `perform`:

```ocaml
module Handler : sig type t : void mod external_ many stateless immutable end
module Safe : sig val perform : Handler.t @ local -> 'a t -> 'a end

module Deep.Safe : sig
  val match_with
    : (Handler.t @ local -> 'c -> 'a) @ unyielding -> 'c -> ('a,'b) handler -> 'b
  val try_with
    : (Handler.t @ local -> 'b -> 'a) @ unyielding -> 'b -> 'a effect_handler -> 'a
  module With_handler : sig ... end   (* handler callbacks also get a token *)
end
(* also Deep.Preemptible.Safe, Shallow.Safe.fiber,
   Shallow.Safe.With_handler, Shallow.Preemptible.Safe.With_handler *)
```

```ocaml
type _ Effect.t += Ask : int Effect.t
let f (h : Effect.Handler.t @ local) () = Effect.Safe.perform h Ask + 1
let r =
  Effect.Deep.Safe.try_with f ()
    { effc = fun (type a) (e : a Effect.t) ->
        match e with
        | Ask -> Some (fun (k : (a, _) Effect.Deep.continuation) ->
                   Effect.Deep.continue k 41)
        | _ -> None }          (* r = 42 *)
```

The token exists for js_of_ocaml double-translation soundness; it does
not prove the effect is handled, and (being `void`) cannot be stored
in ordinary containers.

### Experimental preemption becomes usable (minus-40, #5913)

Builds on the minus-39 groundwork (#4881). **Status: experimental,
default-off, opt-in per fiber, native-only** — the bytecode build
compiles and runs but no `Preemption` is ever delivered (#6333).

New in `Stdlib.Effect`:

```ocaml
type _ Effect.t += Preemption : unit t   (* performed when preempted;
                                            already present at minus-39 *)
type tick_outcome = Preempt | Continue

module Deep.Preemptible : sig
  type ('a, 'b) handler =
    { retc : ...; exnc : ...; effc : ...;
      tickc : unit -> tick_outcome }     (* must be signal-safe *)
  val match_with : ('c -> 'a) -> 'c -> ('a, 'b) handler -> 'b
  val try_with :
    on_tick:(unit -> tick_outcome) -> ('b -> 'a) -> 'b
    -> 'a effect_handler -> 'a
end
module Shallow.Preemptible : sig ... (* continue_with etc. *) end
```

New in `Stdlib.Domain`: `Tick.with_ : interval_usec:int ->
(t @ local -> 'r) @ local once -> 'r` (`value_or_null`-polymorphic in
its result). `Tick.acquire`, `Tick.release`,
`Tick.effective_interval_usec : unit -> int or_null` (the current
tick rate) and the `Preemption` effect already existed at minus-39;
new at minus-40 are `tick_outcome`, the `Preemptible` modules and
`Tick.with_`. Safe variants (`Deep.Preemptible.Safe`, taking an
`Effect.Handler.t`) arrive at ox3.

Run a computation via `Deep.Preemptible.match_with`/`try_with` with a
tick handler, and make the runtime tick via
`Domain.Tick.acquire ~interval_usec` or the scoped `Tick.with_`. On a
tick, handlers are consulted root-fiber-first; the first to return
`Preempt` gets a `Preemption` effect performed in its fiber. Unhandled
`Preemption` effects resume transparently. Allocation-free loops are
only preempted on compilers built with `--enable-poll-insertion` —
which the opam `oxcaml-compiler` package is, so preemption works out
of the box there.

```ocaml
let preemptions = ref 0
let v =
  Domain.Tick.with_ ~interval_usec:1000 (fun _ ->
    Effect.Deep.Preemptible.match_with work ()
      { retc = Fun.id; exnc = raise;
        effc = (fun (type a) (e : a Effect.t) ->
          match e with
          | Effect.Preemption ->
            Some (fun (k : (a, _) Effect.Deep.continuation) ->
              incr preemptions; Effect.Deep.continue k ())
          | _ -> None);
        tickc = (fun () -> Effect.Preempt) })
```

No new public C API: the preemption-related runtime entry points are
internal (and `caml_domain_preempt_self` does not exist — do not
reference it).

### `Stdlib.Dynamic`: dynamically scoped variables (#6412 ox3 … #6978 ox7)

Fiber-aware dynamic binding, safe for parallelism:

```ocaml
(* stdlib/dynamic.mli — @@ portable *)
type 'a t : value mod everything with 'a @@ contended portable
val make : unit -> 'a t
val get : 'a t -> 'a or_null @ contended portable
val with_temporarily : ('b : value_or_null).
  'a t -> 'a @ contended portable ->
  f:(unit -> 'b @ local unique once) @ local once -> 'b @ local unique once
```

```ocaml
let d : int Dynamic.t = Dynamic.make ()
let show () = match Dynamic.get d with Null -> "Null" | This v -> string_of_int v
let () =
  assert (show () = "Null");
  Dynamic.with_temporarily d 1 ~f:(fun () ->
    assert (show () = "1");
    Dynamic.with_temporarily d 2 ~f:(fun () -> assert (show () = "2"));
    assert (show () = "1"));
  assert (show () = "Null")
```

No initial value and no `set`: an unbound dynamic reads `Null`. Values
go in and come out at `contended portable`, so store immutable
or mode-crossing data. Nested bindings shadow and restore (also on
exceptions); a child fiber sees the parent's binding; a continuation
resumed under a different binding reads the *new* binding; a new
thread reads `Null`. Bindings work through C callbacks from ox5.

### Other stdlib changes

- **`Obj.hash_variant : string -> int`** (ox7, #6692):
  `Obj.hash_variant "Foo"` equals ``(Obj.magic `Foo : int)``.
- `Atomic` / `Atomic.Loc`: `set`, `exchange`, `compare_and_set`,
  `compare_exchange` take `('a t [@local_opt])` (ox5); no source
  change needed.
- `Domain.spawn`, `DLS.*`, `TLS.*` lost their explicit
  `@@ nonportable` annotations (ox3, #5532); alerts unchanged.
- `Stdlib_stable.IarrayLabels`' `*_local` functions (`init_local`,
  `map_local`, `mapi_local`, `map2_local`, `combine_local`) are
  `[@zero_alloc]`-compatible (ox2, #6147).
- Module inventories: `stdlib_stable` gains only **`Idx_atomic`**;
  `stdlib_upstream_compatible` is unchanged (`Float_u`, `Int32_u`,
  `Int64_u`, `Nativeint_u`, `mixed_block_layout_v6`); `stdlib_alpha`
  and `stdlib_beta` are still empty. `Stdlib.Modes` and `Multicore`
  are unchanged.

## SIMD

- **Vector primitives on `float array` removed** (ox2, #6128): all
  `%caml_float_array_get128/set128` (and `get256/512`, unaligned `u`
  variants, and every `_indexed_by_int8#/…/nativeint#` variant) on
  **`float array` and `float iarray`** are gone — they can't work
  under `--disable-flat-float-array`. The `floatarray`
  (`%caml_floatarray_*`) and `float# array`
  (`%caml_unboxed_float_array_*`) equivalents remain (MIGRATION step
  10).
- **AVX-512 groundwork** (#6293 ox4; #6294, #6339, #6340, #6676,
  #6708 ox5): the 512-bit vector types (`int8x64`, `int16x32`,
  `int32x16`, `int64x8`, `float16x32`, `float32x16`, `float64x8` and
  their `#` versions) moved from `simd_alpha` to
  **`-extension simd_beta`** (#6676, ox5; `simd_alpha` still enables
  them too), and there are new `mask` / `mask#` types (kind `mask`,
  k-registers). New primitives `%box_mask`, `%unbox_mask`, builtins
  `caml_mask_of_int64` / `caml_int64_of_mask`, and mask load/store
  (`%caml_bytes_getmask`, `%caml_bigstring_setmask`, …).
  `int64x8# array` and `mask# array` work through layout-poly array
  primitives. Backend flags `-favx512dq`, `-favx512cd`, `-favx512bw`,
  `-favx512vl` join the pre-existing `-favx512f` (amd64 only, off by
  default). **There are no AVX-512 intrinsics and no `ocaml_simd`
  512-bit modules yet** — this is types, layouts, load/store and C
  stubs only.
- **`_u` rename**: SIMD type names (`int32x4#`) and primitive-name
  strings (`…_indexed_by_int32#`) are unchanged; only signatures that
  mention `int64#` etc. need respelling (`scale:int64_u`, BMI/POPCNT
  externals, `int128` helpers). `ocaml_simd`'s only API change between
  the two library releases is this respelling; it now declares
  `conflicts: "oxcaml-compiler" {< "5.4.0-ox9"}`.
- **arm64**: 128-bit vectors work natively (NEON, `caml_neon_*`
  builtins); 256-bit vectors compile (lowered); unboxed 512-bit
  vectors and `mask#` are a backend fatal error; an x86 builtin on
  arm64 fails with "External annotated with [@@builtin] is not
  recognized".
- **Fixes**: SIMD registers live across a stack check were not
  preserved over the stack-realloc handler — a silent miscompilation
  with stack checks enabled (ox3, #6334); arm64 unaligned 128-bit
  load/store address computation (ox2, #6388).

## C API

### Upstream 5.3/5.4 (ox1)

- **`caml_result`** (5.3): `caml_callback_res`, `caml_callback2_res`,
  `caml_callback3_res`, `caml_callbackN_res`,
  `caml_process_pending_actions_res`, with `Result_value(v)`,
  `Result_exception(exn)`, `Result_unit`, `caml_result_is_exception`,
  `caml_get_value_or_raise`. The `_exn` variants and
  `Is_exception_result`/`Extract_exception` still compile but are
  obsolete.

  ```c
  caml_result res = caml_callback_res(f, Val_unit);
  if (caml_result_is_exception(res)) { /* cleanup */ }
  return caml_get_value_or_raise(res);
  ```

- **Exception constructors as values** (`fail.h`):
  `caml_exception_failure`, `caml_exception_failure_value`,
  `caml_exception_invalid_argument[_value]`,
  `caml_exception_not_found`, `caml_exception_out_of_memory`,
  `caml_exception_end_of_file`, `caml_exception_zero_divide`,
  `caml_exception_sys_error`, `caml_exception_with_arg[s]`, … for use
  with `caml_raise` or `Result_exception`.
- **Removed**: `caml_failure_exn` (use
  `caml_exception_failure_value`), `caml_raise_async_if_exception`
  (use `caml_get_value_or_raise_async`), `caml_alloc_for_heap`,
  `caml_free_for_heap`, `caml_add_to_heap`,
  `caml_stat_alloc_aligned[_noexc]`.

### OxCaml

- **`OXCAML_RUNTIME`** (minus-40, #6203): `runtime/caml/config.h`
  defines `#define OXCAML_RUNTIME 1` so C stubs can distinguish OxCaml
  from stock OCaml.
- **Stack-allocation regions** (minus-40, #6142):

  ```c
  typedef intnat caml_region_t;
  CAMLextern caml_region_t caml_region_begin(void);
  CAMLextern void caml_region_end(caml_region_t);
  ```

  Lets C code open/close a region so stack allocations made by OCaml
  callbacks returning `local` values are freed at `caml_region_end`.
  At region end there must be no remaining GC-root reachable
  references into the region (clear `CAMLlocal` roots to `Val_unit`
  first). Failing to end a region is fine — the next
  `caml_region_end` closes all regions begun after it, so no need to
  call it before `caml_raise`.
- **Heap extension** (ox6, #6248) — donate memory already laid out as
  OCaml blocks to the major heap:

  ```c
  /* caml/memory.h */
  CAMLextern void caml_add_blocks_to_heap(void *base, size_t size,
                                          void (*free_callback)(void *, size_t));
  ```

  ```c
  value *base = malloc(3 * sizeof(value));
  base[0] = Make_header(2, 0, 0);          /* from <caml/gc.h> */
  base[1] = Val_long(1); base[2] = Val_long(2);
  caml_add_blocks_to_heap(base, 3 * sizeof(value), free_cb);
  return Val_hp(base);                     /* an ordinary (int * int) */
  ```

  Each block must have wosize ≥ 1; `free_callback` runs during GC
  (finaliser restrictions apply) once every block in the extent is
  dead.
- **Atom symbols** (ox6, #6779): the runtime exports
  `caml_atom_0` … `caml_atom_255` (`Atom(0) == Val_hp(&caml_atom_0)`).
- **AVX-512**: `caml/simd.h` gains `Vec512_val{,d,i}`,
  `Store_vec512_val{,d,i}`, `caml_copy_vec512{,d,i}`.
- Atomic-field entry points: see [Atomic fields](#atomic-fields).
- Unchanged and still valid: `Assert_mixed_block_layout_v6`, the
  `caml_alloc_local*` family.
- Internal (`CAML_INTERNALS`) churn: `caml_gc_message` lost its
  category argument (use the `CAML_GC_MESSAGE(CATEGORY, fmt, …)`
  macro); frame descriptors have a new compact encoding (ox6, #6399;
  ~40% smaller frametables) — code walking `frame_descr` by hand must
  use `caml_decode_frame_descr`.

## GC and runtime

- **Runtime 4 retired** (ox2, #6218): `runtime4/` (~60k lines),
  `otherlibs/systhreads4/`, and `debugger4/` are deleted.
  `./configure --disable-runtime5` is a **hard error** ("OxCaml 5.4
  does not support runtime4 and must be configured with
  --enable-runtime5"); `--enable-runtime5` is accepted as a no-op.
  `Config.runtime5` still exists and is always `true`; the `%runtime5`
  primitive always evaluates true. ocamltest predicates
  `runtime4`/`runtime5` were removed — delete them from TEST blocks.
- **Idle-phase major GC** (ox6, #6484; upstream backport): after
  sweeping, the major GC idles until allocation passes
  `small_heap_limit` words (default 262144). Tune with
  `OCAMLRUNPARAM=Xsmall_heap_limit=128M` (a count of *words*, so
  1 GiB) or `Gc.Tweak.set "small_heap_limit" n`.
  `OCAMLRUNPARAM=Xhelp=1` lists all tweaks.
- `OCAMLRUNPARAM=Xmeasure_frametables=1` (ox4, #6487) prints
  frametable statistics at startup.
- **Frametables go to `.rodata`** (minus-40, #6094). The
  `-frametables-in-rodata` / `-no-frametables-in-rodata` flags this PR
  added were **removed again at ox6** (#6399) — don't pass them.
- minus-40: `Gc.quick_stat` is faster (#6139; a skipped-domain bug was
  fixed by #6164 before the tag). `OCAMLRUNPARAM=d=<too-large>` no
  longer aborts; the value is clamped (#6215).
- ox2 systhreads/preemption stability fixes: Tick handle released on
  normal thread termination (#6331), division-by-zero guard in the
  tick hook (#6329), no preemption continuation in the bytecode
  runtime (#6333), zombie-thread fix (#6345), dynamic-binding cache
  flushed on async unwind across fibers (#6341).
- GC pacing fix for large off-heap (bigarray/custom) data (ox5,
  #6749); many-domain contention fixes (ox5, #6719).
- Fixes (ox3–ox9): bigarray proxy creation race (#6344), bigarray byte
  indexing by a truncated `nativeint`/`int64` (#7059), ephemeron
  memory corruption (#6323), signal-handler race (#6342), effects + C
  callbacks + locals root scanning (#5747), tick-handler stack-growth
  use-after-free (#6764), missing poll points in loops that only
  allocate locally (#6469), leak when tail-calling a local-returning
  function (#6396).

## Compiler flags and driver

### Added

| Flag | Tools | Tag | Meaning |
|------|-------|-----|---------|
| `-flambda2-match-in-match` | — | minus-40 (#4374) | Match-in-match optimisation (default off). |
| `-dunique-ids`, `-dno-unique-ids`, `-dlocations`, `-dno-locations` | `ocamlnat`, `expect.opt` | minus-40 (#6159) | Newly accepted by these tools. |
| `-keywords <version>[+kw]` | compilers, ocamldep | ox1 (5.4 merge) | Select the keyword edition, e.g. `-keywords 5.2` to un-reserve `effect`. |
| `-i-variance` | — | ox1 (5.4 merge) | With `-i`, print inferred variance/injectivity: `type +!'a t = A of 'a`. |
| `-open-cmi <file.cmi>` | `ocamlopt`/`ocamlc`/`ocaml` only — **not** `ocamldep` | ox2 (#6013) | Like `-open`, but reads the signature from the given `.cmi` rather than resolving a module on the include path. |
| `-experimental-optimizations` | — | ox2 (#6174) | Enable a bundle of experimental codegen optimizations not yet on by default (currently: prologue shrink-wrap, x86 peephole, `SPLIT_AROUND_LOOPS`, `AFFINITY`, CFG merge-blocks / value-propagation / dead-trap-handler elimination, …). Subject to change; also an `OCAMLPARAM` key. |
| `-dbranch-relaxation-max-displacement <n>` | — | ox2 (#6309) | Debug-only branch-relaxation testing knob. |
| `-omit-leaf-frame-pointers` / `-no-…` | ocamlopt | ox5 (#6679) | On frame-pointer builds, skip the frame in leaf functions (amd64; default off). |
| `-cfg-block-layout` / `-no-…` | ocamlopt | ox7 (#6918) | Static CFG block reordering for fallthrough / cold-block placement (default off). |
| `-functorize` | ocamlopt, ocamlc | ox6 (#5903) | Bundle parameterised modules into a single ordinary functor (see below). |
| `-gno-ocamldebug-types` | ocamlc only | ox5 (#6706) | Drop ocamldebug typing info from `.cmo`/`.cma` (much smaller `-g` bytecode objects). |
| `-dissector-max-linker-parallelism <n>` | ocamlopt | ox4 (#6447) | Cap parallel partial links in the `-dissector` pass. |
| `ocamldep -modules-only` | ocamldep | ox4 (#6540) | Like `-modules` without the leading `file:`. |
| `-ddebug-avail-sets`, `-flambda2-simplify-stubs`, `-no-x86-peephole-remove-redundant-{extension,test}`, `-X flambda2-meet-strategy=…`, `-X flambda2-combine-inlining-arguments=…`, `-X argument-types-useful=…` | ocamlopt | ox3–ox6 | Debug/expert knobs. |

New env vars (ox2, #5651): `OPT_FUEL_RECORD` / `OPT_FUEL_THRESHOLD` /
`OPT_FUEL_FAIL` for bisecting which optimization step miscompiles
(with `tools/fuel_bisect.py`). New configure option (ox1):
`--enable-keyword-edition=X.Y[+kw]` (see the `effect` keyword above).

### Removed / renamed (hard errors if passed)

- `-flambda2-expert-cont-specialization-budget <int>` →
  **`-flambda2-expert-cont-specialization-threshold <float>`**
  (minus-40, #4374).
- `-gdwarf-config-max-type-to-shape-depth` →
  **`-type-to-shape-max-depth`** (ox6, #5103).
- `-frametables-in-rodata` / `-no-frametables-in-rodata` (existed only
  minus-40..ox5) removed (ox6, #6399).

### Default flips and behaviour

- Continuation lifting/specialisation is on at `-O2`/`-O3` (minus-40,
  #4374).
- `-cfg-eliminate-dead-trap-handlers` is **on** by default (ox2,
  #6267).
- **`-shape-format` defaults to `debugging-shapes`** unconditionally
  (ox6, #5103; was `old-merlin`). Affects `.cms`/`.cmt` contents;
  `-shape-format old-merlin` restores the old format (the `-help` text
  is stale on this).
- DWARF fidelity presets are far more generous at every level (ox6,
  #5199); `-gdwarf-compression` now also applies to plain `-g`
  (#6490).
- `-dprofile` works with `-stop-after middle_end` (#6550).
- Flat-mangled **data** symbols changed shape (ox6, #6405):
  `camlFoo__add_4` → `camlFoo__add_0_1` (code symbols unchanged).
  Name mangling is still `flat` by default.
- Better error on corrupted `.cmxa` files (ox2, #6221); O(N²) linker
  module-tracking fixed — faster large links (ox2, #5807).
- The compiler itself is built with `-O3` (ox2, #5778) — a faster
  compiler binary; no change to user flags.

### Language extensions

- New: `mode_polymorphism`, `mode_polymorphism_beta`,
  `mode_polymorphism_alpha`, `mode_polymorphism_printing` (ox7,
  #5483) — only the `_alpha` level enables anything.
- AVX-512 vector types moved from `simd_alpha` to **`simd_beta`**
  (ox5, #6676).
- Layout polymorphism (`let poly_`/`val poly_`) needs
  `layout_poly_alpha`; `any` fields need `layouts_beta`.
- Default-enabled set otherwise unchanged; `comprehensions` is still
  beta, `runtime_metaprogramming` still beta and native-only.

### `-functorize` (ox6, #5903)

Bundles parameterised modules (`-as-parameter` / `-parameter` /
`-as-argument-for`, all pre-existing) into one ordinary functor:

```
ocamlopt -c -as-parameter p.mli
ocamlopt -c -as-argument-for P p_int.mli p_int.ml
ocamlopt -c -parameter P foo.mli foo.ml
ocamlopt -functorize Foo -o bundle.cmx
# main.ml: let module Inst = Bundle.Make (P_int) () in Inst.Foo.show ()
```

## compiler-libs and tooling

### `compiler-libs.common` split (ox5, #6470)

`ocamlcommon` (`compiler-libs.common`) now contains only utils and
parsing (`Ast_helper`, `Ast_mapper`, `Parsetree` helpers, `Location`,
`Longident`, `Lexer`, `Parse`, `Pprintast`, `Clflags`, `Config`,
`Warnings`, `Misc`, `Load_path`, `Ident`, `Compilation_unit`, …).
Everything typing/lambda/driver moved to the new
**`compiler-libs.frontend`** (`ocamlfrontend`): `Path`, `Types`,
`Btype`, `Ctype`, `Env`, `Typedtree`, `Typecore`, `Typemod`,
`Printtyp`, `Out_type`, `Predef`, `Subst`, `Shape`, `Cmi_format`,
`Cmt_format`, `Cms_format`, `Tast_iterator`, `Tast_mapper`,
`Untypeast`, `Mode`, `Jkind`, `Lambda`, `Compmisc`, `Compenv`,
`Main_args`, `Persistent_env`, `Primitive`, …

```
Error: No implementations provided for the following modules:
         Path referenced from a.cmx
```

Fix: add `compiler-libs.frontend` to `(libraries …)` (MIGRATION step
12). `compiler-libs.bytecomp`/`.optcomp` already depend on it; pure
parsetree ppx code needs nothing.

Other compiler-libs API changes:

- `oxcaml_utils` library removed (ox3, #6450): `Doubly_linked_list`,
  `File_sections`, `Lru` are now top-level modules in `ocamlcommon`.
- `Compmisc.read_clflags_from_env` → `Location.read_clflags_from_env`;
  `Clflags.open_modules` → `Clflags.open_args` (`Open of string |
  Open_cmi of string`); `Clflags.transparent_modules` →
  `no_alias_deps`.
- `Compiler_hooks.pass` gained a second type parameter (ox7, #6826):
  `Parse_tree_intf`/`Parse_tree_impl` hooks now *return* a (possibly
  rewritten) AST.
- Warning constructors renamed with the warnings above
  (`Mod_by_top` → `Redundant_modifier`, …).
- Parsetree jkind-annotation shape changed (ox2, #6367).

### Tools and debugging

- **`ocamlfilt`** (ox2, #5100): a pure-OCaml symbol demangler,
  installed to bin (`ocamlfilt` / `ocamlfilt.byte`). Handles pre-5.3
  flat, post-5.3 flat, and structured mangling schemes with
  auto-detection (`--format auto|flat0|flat1|structured`); reads
  symbols from argv or stdin. It only demangles a line that is
  **exactly one symbol**, so `nm x.o | ocamlfilt` does nothing —
  extract the symbol column first:
  `nm x.o | awk '{print $3}' | ocamlfilt`.

  ```
  $ ocamlfilt camlFoo__add_0_4_code _CamlU3FooM5InnerF5add_0_4_code
  Foo.add_0_4_code
  Foo.Inner.add_0
  ```
- **OxCaml LLDB** is now `21.1.0+oxcaml1` (ox3, #6384), which
  demangles the structured scheme.
- DWARF (with `-g -gno-upstream-dwarf`): phantom lets survive to
  Linear so optimised-out variables print (#6517), more pattern-match
  debug info (#6354), correct `DW_AT_call_file/line/column` (#6877),
  correct paths for inlined cross-directory functions (#6882), type
  shapes for `any` fields (#6403), much faster generation (#5124).
- Profiler support (ox2): effect implementation offsets emitted as ELF
  notes (#6274) and fiber ids recorded for magic-trace (#6032).
- `.cmi` stamp normalisation (ox5, #6542): fewer spurious rebuilds.
- Saved `.cmir-cfg`/`.cmir-linear` files carry a configuration
  fingerprint (ox7, #6908).
- Merlin: destruct for unboxed tuples (#6849), `ocaml-index`
  ppx-aware staleness (#6654); `eval.mli` is installed with the
  compiler (ox2, #6207), fixing jump-to-definition for the
  metaprogramming `Eval` library.
- Attributes on module aliases in signatures now parse (ox3, #5869):
  `module S = String [@foo]`.

### Metaprogramming (still beta, native-only)

Surface unchanged from minus-39; fixes and completeness only:
first-class modules in quotes (ox3, #5884), unboxed literals in quotes
(ox4, #6650), mode constraints in quotes no longer dropped (#6647).
Recipe:

```ocaml
(* q.ml *)
#syntax quotations on
let power n : <[int -> int]> expr =
  let rec go n (x : <[int]> expr) : <[int]> expr =
    if n = 0 then <[ 1 ]> else <[ $x * $(go (n - 1) x) ]>
  in
  <[ fun x -> $(go n <[ x ]>) ]>
let () = Printf.printf "%d\n" ((Eval.eval (power 3)) 5)   (* 125 *)
```

```
ocamlopt -extension runtime_metaprogramming -uses-metaprogramming eval.cmxa q.ml
```

## Jane Street libraries (preview.130.106+341 → preview.133.03+642)

Not part of the compiler, but shipped in lockstep; OxCaml-relevant
changes only. Old → new spellings for every rename and removal below:
table in MIGRATION step 11.

- **Lockstep**: every `133.03+642` package (not just `parallel`)
  conflicts with compilers before ox9 and every `130.106+341` package
  with every compiler other than minus-39 (MIGRATION step 0). New
  packages `await_in_async`, `concurrent_kernel`,
  `portable_intrusive_stack`, `unboxed_indexing`.
- **`unboxed`**: `Int32_u.t`, `Int64_u.t`, `Nativeint_u.t`,
  `Float32_u.t` are now `int32_u`, `int64_u`, `nativeint_u`,
  `float32_u` (`Float_u.t = float#` unchanged). Removed: the `Array`,
  `Array_index` and `Iarray_index` submodules of `Int64_u` /
  `Int32_u` / `Nativeint_u` (replaced by layout-polymorphic
  `Base.Array`, `[@kind …]` instances and the new `unboxed_indexing`
  package).
- **`base`**: `Base.Export` dropped `sexp_of_int64_u` & co. (breaks
  derivers on bare `int64_u` fields). Renamed: `Hashtbl.choose__local`
  → `choose_local`. New: `Or_error.Portable`,
  `Modes.Many_via_aliased`/`Aliased_via_many`, `min__local`/
  `max__local` on the scalar modules (only `Float` had them before),
  `Array.get_or_null`, `Hashtbl.find_and_remove_or_null`,
  `Map.update_or_null`, `Option.merge__stack`,
  `Or_error.try_with__stack`, `Iarray.zip__stack`. Not a break:
  `Comparable.clamp` is now declared as `(t, Error.t) Result.t`, which
  is the same type as the old `t Or_error.t`.
- **`capsule`/`capsule0`**: the low-level `Capsule.Prim` API collapsed
  its `*_unique`/`*_once`/`*_local` variants into ppx_template mode
  instances, and `Key.box`/`unbox` lost their unsuffixed instance.
  `Blocking_sync` callbacks now also require `@ unyielding`. New
  `Data.Or_null`, `Portable_via_contended`.
- **`portable`**: `Atomic.get`/`set` are mode-templated over
  visibility (`read_write`/`read`, `write`); `Atomic.make` returns
  `@ unique`.
- **`basement`**: new `Stdlib_shim.Dynamic`, `Stdlib_shim.Effect` (the
  safe-effects API); `Stdlib_shim.runtime5` removed;
  `Primitives.box_int64` & co. became `Primitives.Int64_u.box`.
- **`core`**: `Or_error.Portable`, `Set_once.Atomic`,
  `bin_writer_t__local` from `[@@deriving bin_io ~localize]`.
- **`ppx_template`**: language unchanged.
- **ppx + effect syntax**: `| effect E, k ->` patterns fail in any
  file preprocessed by a ppx ("migration error: effect patterns is not
  supported before OCaml 5.02").

## Compiler internals (condensed)

- **Flambda 2**: bottom-environment handling in join/reaper (#6348,
  #6349), poison values in terms (#6273), CSE of immutable array
  loads (#6300), switch-arg equations (#6373), classic-mode
  approximation fix for tagged immediates (#6283), inlining counts
  calls as removed operations (#5991), `Map.find_or_null` adoption
  (#6290). The "meet for array element kinds" fix (#5771) remains
  net-reverted (by #6220).
- **ikinds**: recursive payload ikinds (#6199), temporary type
  declarations (#6201), out-of-fuel note suppressed for ikind errors
  (#6197), nondep jkind bug (#6271).
- **Backend**: arm64 branch-relaxation fixes (#6309, #6311, #6388),
  unaligned 128-bit load/store on arm64 (#6388), monotone backwards
  jumps in the x86 binary emitter (#6025), ud2 guards for data in
  text (#6129), ZMM28 save/restore (#6252), macOS/amd64 runtime
  symbol fix (#6320).
- **Representation plumbing**: `value_kind` recomputation for
  variable-representation constructors (#6307), inline records with
  `any` fields (#6240), layout bug for aliases in matching (#6152),
  `Punspecializedarray` array kind (#6101), small numbers as hidden
  env types (#6232).
- **minus-40 internals**: normal-form morphisms (#5453), n-way join
  fixes (#6170), cfg-merge-blocks equality fix (#6223), fexpr
  completions (#6122, #5837), `%obj_xxx` primitive cleanup (#6080).
- **Docs** (`jane/doc`): kind-syntax grammar refresh — `void`,
  `bits8/16`, `vec256/512`, `*_or_null` abbreviations, `any_non_null`
  removed (#6353); borrow-vs-stack-allocation interactions documented
  in `_07-uniqueness/borrow.md` (#6160): `exclave_` values cannot
  escape a `borrow_` region; implicit-kinds doc updated for
  structures (#6114).
- **ox3–ox9 internals**: Flambda 2 BFS meet strategy (#6191),
  improved meet for naked immediates (#4394), reaper handles boxed
  numbers (#6437) and can no longer be run under `-Oclassic` (#6993
  reverted by #7067), cmx loading avoided for non-inlinable
  applications (#6258), speculative-inlining code-size fixes (#6705);
  x86 codegen without flags — `neg` (#6701), `test` for booleans
  (#6552), csel/cmov (#6659), immediate folding (#6704), lea folding
  (#6680), redundant extension/test removal (#6703, #6722); compact
  frametables (#6399, #6488); CFG invariants (#6371), merge blocks
  after stack checks (#6465), static block layout (#6918); x86 binary
  emitter relocation handling (#6009); dissector parallel partial
  links (#6447, #5824); DWARF DIE sharing/compact forms (#6945,
  #6922); mode solver: mode variable levels (#5478), subsumption
  (#6279), functional mode polymorphism (#5483); ikind provenance in
  errors (#6262, #6769); `ocamlfrontend` library split (#6470);
  normalised `.cmi` stamps (#6542); arm64 relocation constraints (the
  ox9 commit), `%progbits` on ARM (#6389).
