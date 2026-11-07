# OxCaml Changes: 5.2.0minus-25 to 5.2.0minus-31

This document summarizes the key changes between OxCaml versions 5.2.0minus-25
and 5.2.0minus-31.

## Major New Language Features

### Unboxed Booleans (#5166)

New primitive `bool#` type with `#false` and `#true` literals. Fully
pattern-matchable and usable without allocation:

```ocaml
let classify (b : bool#) : int =
  match b with
  | #false -> 0
  | #true  -> 1

type flag = { enabled : bool#; count : int }
```

Kind: `bits8 mod everything` (same as `int8#`).

### Unboxed Unit (#5151, #5156)

New `unit#` type with `#()` literal, kind `void mod everything`. It has no
runtime representation at all — useful as a placeholder in generic code, as
filler in kind products, and for eliminating final-continuation overhead:

```ocaml
let ignore_u (x : unit#) : int =
  match x with
  | #() -> 0

let noop () : unit# = #()
```

### Simple Borrowing (`borrow_`) (#5215)

New `borrow_` keyword: a prefix expression form (`borrow_ e`) that cooperates
with the uniqueness analysis. It is valid in exactly three positions —
anywhere else raises "The borrow_ operator must appear directly in a valid
borrowing context" (`Borrowed_out_of_context`):

```ocaml
let r = ref 0 in
f (borrow_ r);           (* function argument *)
let y = borrow_ r in ... (* let-binding RHS *)
match borrow_ r with _ -> ... (* match scrutinee *)
```

Two new diagnostics:

- **Warning 216 `Use_during_borrowing`** (short name `use-during-borrowing`,
  description: "Use of a value during an active borrow.") is raised when a
  value is used while being borrowed.
- **Error `Unique_use_during_borrowing`** is raised by the uniqueness
  analysis when a unique use conflicts with an active borrow; it carries
  the region location, borrow occurrence, and a `cannot_force` reason.

The error reporting shows "borrowed" instead of generic "used" when explicit
borrows conflict with unique usage.

### Implicit Kind Declarations (#4285)

Signatures can declare that specific type-variable names default to a chosen
kind via a floating attribute `[@@@implicit_kind: ...]`:

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

Multiple bindings in one attribute use product syntax:

```ocaml
[@@@implicit_kind: ('a : immediate) * ('b : immediate)]
val swap : 'a * 'b -> 'b * 'a
```

Rules (from `jane/doc/extensions/_06-kinds/implicit.md`):

- **Cannot be overridden**: a variable declared with an implicit kind must
  keep that kind; narrowing or changing it is an error.
- **Inherited by nested signatures**: a nested `module type` or `sig`
  inside the same module type sees the outer implicit kind.
- **Cannot be re-declared** in a nested signature (error: "The implicit
  kind for ... is already defined at ...").
- **Not propagated through `include`**: `include S` does not carry over
  `S`'s implicit kinds into the including signature.
- **Cannot be declared inside structures** (signatures only).
- **Affects `constraint` clauses** — in fact, implicit kinds are
  currently the only way to set a `constraint` to certain kind values.

### Abstract Kinds and Kind Aliases (#5070, #5071)

The compiler now supports abstract kinds and first-class
kind aliases as part of a broader kind-system refactor. User-visible effect:
the `kind_abbrev_` keyword has been **renamed to `kind_`** and the parsetree
`Psig_kind_abbrev` / `Pstr_kind_abbrev` are now `Psig_jkind` / `Pstr_jkind`
carrying a `jkind_declaration` record.

### Bootstrap Toolchain is now OCaml 5.4.0 (#5036, #5216)

The OxCaml runtime version remains `5.2.0+ox` and the the opam package is still
`ocaml-variants.5.2.0+oxcaml`.

What changed is the bootstrap: building OxCaml now requires a working OCaml
`5.4.0` compiler on `PATH` (previously `4.14.1`) Alongside the bootstrap
change, selected upstream 5.4.0 features were backported onto the 5.2.0+ox
base:

- `Format_doc` (PRs #5357, #5378, #5380, #5381 — backporting upstream
  #13169, #13311, #13365, #13487).
- `ocamlformat` 0.28.1 (#5180, #5181, #5205, #5211) with a tree-wide
  reformat.
- Nix flake refresh for the new toolchain (#5194).
- `caml_make_vect` → `caml_array_make` runtime symbol rename
  (#5339, backporting upstream #13003).

### Polymorphic Let-Bindings (`let poly_`, `val poly_`) (#5392)

New parser syntax for explicitly polymorphic let-bindings and value
declarations, gated on the `Layout_poly` alpha extension. The flag is recorded
in the parsetree (`pvb_is_poly`, `pval_poly`). The typechecker currently
reports a "not yet implemented" error when it encounters the annotation,
but the parser accepts it. Because `poly_` attaches to idents (not the
whole `let`), mixed forms like `let poly_ x and poly_ y in ...` are valid.

### Representation Types (`repr_`) (#5068)

New `repr_` keyword introduces representation types in the parser/typechecker
(`Ptyp_repr` / `Ttyp_repr`), plus a `Trepr` case in `type_expr`. Gated on the
`Layout_poly` alpha extension. Disallowed in record fields.

---

## Mode / Kind / Modality System Changes

### Non-`value` Base Kinds Imply `mod external_` (#5162)

Kinds `bits8`, `bits16`, `bits32`, `bits64`, `float32`, `float64`,
`untagged_immediate`, `vec128`, `vec256`, `vec512`, `void`, and `word` now
implicitly include `mod external_` (in fact by `minus-31` they mode-cross
everything — the compiler prints e.g. `bits8 mod everything`). This enables
write-barrier elision.

To get the unmoded version, append `_internal`:

```ocaml
(* These are equivalent *)
type t : bits64
type t : bits64_internal mod external_
```

### New `-kind-verbosity` Flag (#5397)

Controls jkind printing verbosity (the underlying verbosity option was
added for Merlin in #5304). Related rendering fix in #5398.

### Mode Hints on Modules and Modalities (#5034, #5183)

Mode hints now apply at module and modality positions, not just expressions.

### Per-Axis Default Modalities in Signatures (#5290)

Default modalities in signatures now combine with explicit ones per-axis
rather than all-or-nothing.

### `include functor` Signatures (#5306)

Signatures generated from `include functor` now always zap modalities to
their strongest form (the floor). Affects library authors who expose
functor-generated signatures.

### Functors with Non-Legacy Modes (#5107, #5230)

Functors can now have mode-annotated input and output, and currying a functor
correctly constrains modes.

### Separability Moved into the Layouts Extension (#5231)

`separable` no longer requires a dedicated extension flag — it is part of
the core `layouts` extension.

### Uniqueness Analysis Refactor (#5302)

`here_occurs` and `access_order` were merged into a single `usage_order`
type with variants `Seq_before | Seq_after | Par`. Error messages now say
"used as unique" consistently and use "at:" when pointing at prior usage.

### Stack-Allocated `bytes` (#5366)

`Bytes.create__stack : int -> bytes @ local` lets you allocate bytes on the
stack. Many stdlib functions (`Bytes.blit`, `Bytes.blit_string`,
`String.blit`, and various `index_from`/`rindex_from` variants) now accept
`@ local` arguments.

### Documentation Clarifications (#5274)

`jane/doc/extensions/_05-modes/intro.md` emphasises that mode-crossing is a
type-system property distinct from "not allocated", and that stack allocation
is a *may*, not a *will*.

### Miscellaneous

- Mode solver bug fix (#5233).
- Add actual mode to `Texp_ident` and mode/modality locations to the typedtree
  (#5218, #5297, #5300).
- `printtyp` no longer raises "undefined modalities" on value descriptions
  (#5382).
- Improved mode-error formatting with print-box indentation (#5275).
- Environment scoping fix for jkind errors (#5173).
- Stage checks for record and unboxed record labels run properly (#5158).

---

## Unboxed Types / Layouts / Small Numbers

### Mixed Block Layout v5 (#5324)

Block indices into mixed products now use a **52-bit offset and 12-bit gap**
(previously 48/16). One practical consequence: indices into records that
mix values and non-values, occupying over **2^12 bytes** (previously 2^16),
cannot be created. The mixed block version bumped from v4 to v5:

- `Stdlib_upstream_compatible.mixed_block_layout_v4` →
  `mixed_block_layout_v5`.
- C macro `Assert_mixed_block_layout_v4` → `Assert_mixed_block_layout_v5`.

### `value_or_null` is now a first-class base layout

Previously, `value_or_null` was alpha-only — it was displayed as `value`
and couldn't appear in jkind annotations unless `-extension-universe alpha`
was set. The doc at `jane/doc/extensions/_03-unboxed-types/01-intro.md` now
lists it among the base layouts with no alpha caveat:

> `value_or_null` is a superlayout of `value` including normal OCaml values
> and null pointers.

Related: the `any_non_null` layout has been removed from the user-facing
layout list (it was also alpha-only before). `any` no longer carries the
compat-special-case note about being interpreted as `any_non_null` for
backwards compatibility with arrays.

### `void`, `bits8`, `bits16` base layouts documented

The base-layouts list in the intro doc now explicitly includes `void`
(layout of `unit#`), `bits8` (layout of `int8#`), and `bits16` (layout
of `int16#`).

### Small-Int Indexing and Bigstring/Bytes Primitives (#4779)

Arrays/strings/bigstrings/bytes can now store `int8`, `int16#` values and be
indexed by `int8#` / `int16#`. New sign-extending primitives:

- `%caml_bytes_geti8`, `%caml_bytes_geti16`
- `get8` / `set8` / `set16` family with `*_indexed_by_*` variants.

### Small-Int Bit Intrinsics (#5393, #5407)

- `ctz`, `clz`, `popcnt` for `int8#` and `int16#`.
- `caml_popcnt_int16`, `caml_lzcnt_int16`, `caml_bmi_tzcnt_int16` (x86 via
  the 0x66 operand-size prefix).

### More Small-Int Conversions (#5305)

Extended conversion primitives between small-int types.

### Other Layout Changes

- Arrays of unboxed pairs of `vec128` now allowed (#5239).
- Wide vectors on arm64 lowered to tuples (#5291, bridged by the
  `Preinterpret_boxed_vector_as_tuple` / `Preinterpret_tuple_as_boxed_vector`
  primitives).
- Boxed vectors use tag 0 on amd64 (#5410) — a visible runtime tag change.
- `Bigarray.Genarray.t` and friends now have `any`-kinded type parameters
  (#5135).
- Fix `%unsafe_ptr_set` with NULL base (#5087).
- Several fixes to symbol projections involving unboxed numbers (#5299).

---

## SIMD

- **int16 x86 intrinsics** (#5407), instruction tables generated by the `simdgen` tool; adds 0x66 operand
  size prefix support.
- **Wide vectors** on arm64 lowered as tuples (#5291).
- **Unboxed pairs of `vec128` inside arrays** (#5239).
- **`ymm` save bug** on amd64 fixed (#5232).
- **Fourth register class** for `caml_call_gc_sse` (#5136): old `caml_call_gc`
  renamed; a zero-SIMD variant is now available. Lays groundwork for
  preemption-related changes.

---

## Stdlib / stdlib_stable / Library Additions

### New / Changed Values

- `Sys.io_buffer_size : int` (#5078) — size of runtime IO buffers.
- `Bytes.create__stack : int -> bytes @ local` (#5366).
- `Bytes.blit` / `Bytes.blit_string` / `String.blit` / various index
  functions now take `@ local` arguments (#5366).
- `Obj.raw_field` / `Obj.set_raw_field` are now regular `val`s rather than
  `external`s and have been optimised (#5241). Float fields print as floats
  in debug output.
- `Sys.getenv_opt` is now a dedicated non-raising external
  (`caml_sys_getenv_opt`) and no longer trashes backtraces (#5076).

### GC Stats Overhaul (#5226)

`Gc.stat` and `Gc.quick_stat` have much richer and more accurate output in
runtime5. `quick_stat` now returns end-of-last-cycle values rather than
zeros for most fields; `heap_chunks` is populated in runtime5. The
documentation has been rewritten accordingly.

### Iarray Gets layout_poly + or_null Support (#5309, #4532)

`Stdlib_stable.Iarray` / `IarrayLabels` functions are now quantified over
broader kinds:
- `length`, `get`, `( .:() )` are `[@@layout_poly]` over `('a : any mod separable)`.
- `init`, `append`, `concat`, `sub`, `to_list`, `of_list`, `to_array`,
  `of_array`, `iter`, `iteri`, `map`, etc. are quantified over
  `('a : value_or_null mod separable)`.

### Array / ArrayLabels Separability (#5339)

Externals and many functions now quantified over
`('a : value_or_null mod separable)`. Stdlib primitive bindings switch to
the upstream-aligned runtime symbols:

- `Array.make` now binds to `caml_array_make` (was `caml_make_vect`).
- `Array.create_float` now binds to `caml_array_create_float` (was
  `caml_make_float_vect`).

The old C symbols `caml_make_vect` and `caml_make_float_vect` are kept as
thin back-compat shims in `runtime/array.c` that forward to the new ones,
so hand-written external C code using the old names continues to link.
Error messages from the zero-alloc checker show the new names.

### Effect Module Refactor (#4901)

Major internal cleanup. `%resume` now takes a continuation directly;
`caml_alloc_stack` + `%runstack` merged into a new `%with_stack` primitive
(plus `%with_stack_bind` for dynamic bindings). Bytecode gains a new
RUNSTACK-family instruction.

The **public** `Effect.continuation` type is unchanged — it is still
`('a, 'b) continuation`. However, the **internal** GADT `cont` type
gained an extra parameter (now `('a, 'x, 'b) cont` in the Deep module,
`('a, 'b, 'x) cont` in Shallow, with `'x` carrying the fiber's
termination type) and is wrapped in the public `continuation` via
`[@@unboxed]`. Code that uses only the documented `Effect.Deep` /
`Effect.Shallow` API (`('a, 'b) continuation`, `continue`,
`discontinue`) does not need to change; code that directly uses
`%resume`, `%runstack`, or `caml_alloc_stack` must migrate to
`%with_stack` / `%with_stack_bind`.
The `Must_not_enter_gc` internal module has been removed.

### Thread-Local Dynamic Bindings Removed (#5171)

The dynamic-variable runtime API is now just `make`, `get`, and
`with_temporarily`. Per-thread dynamic-root semantics was unsupported by
most users and is now handled exclusively in `Basement.Dynamic`.

---

## Parallelism / Capsules / Runtime

### `%domain_index` Primitive (#5312)

Returns the current domain's index as an untagged int
(`Pdomain_index`). Wired through runtime4, runtime5, and arm64/amd64
codegen.

### Fibers Return to Allocating Domain (#5363)

Addresses a class of issues around multi-domain fiber cleanup.

### Manual Module Initialization (#5395)

New flags `-manual-module-init` / `-no-manual-module-init`. When enabled, the
compiler emits a unit dependency table and a manual init routine with proper
ordering, delayed frametable registration, caught-exception reporting, and
idempotent initialisation. Useful for embedding and staged startup.

### Config Reports runtime5 (#5415)

`ocamlc -config` now reports `runtime5`, making the runtime variant
programmatically visible.

---

## Runtime Metaprogramming / Quotes

Significant internal work on the quotation / runtime-metaprogramming
system (`CamlinternalQuote`, `Translquote`): type-information
inspection, polymorphic applications, record/variant disambiguation,
and borrow support under quotes (#5090, #5214, #5154, #5094, #5215),
plus assorted fixes (#5083, #5319). Still experimental in this window.

---

## Flambda 2 / Optimizer / Codegen (internal — condensed)

All internal; no source changes needed. Highlights a user might notice:

- Inlining heuristics tweaked (#5093, #5119); many reaper crash fixes
  (#5129 … #5379); simplify-terminator hardened (#5174, #5389).
- **Bit-matrix interference graph for IRC** (#5296) with a new
  `-regalloc-param BIT_MATRIX_THRESHOLD:k` knob.
- arm64: typed DSL + binary emitter (#5193, #5177); fix for large
  stack offsets (#5130). 64-bit `.eh_frame*` support (#5303).
- `noalloc` C calls no longer switch stacks in no-stack-check builds
  (#5224); in stack-check builds the OCaml stack pointer moves from
  `rbx` to `r13` (visible in debuggers/profilers).

---

## New Compiler Flags

Oxcaml-specific flags live in `driver/oxcaml_args.ml`; the upstream
`-kind-verbosity` lives in `driver/main_args.ml`.

| Flag | Description |
|------|-------------|
| `-kind-verbosity <int>` | Control jkind printing verbosity (#5397) |
| `-manual-module-init` / `-no-manual-module-init` | Emit the unit-dependency table for manual module initialisation. Default: off. (#5395) |
| `-verify-binary-emitter` | arm64 only: verify binary emitter output matches the system assembler; exits on mismatch (#5198) |
| `-dissector-assume-lld-without-64-bit-eh-frames` / `-no-...` | Assume the linker is LLD without 64-bit eh_frame support (default on) (#5303) |
| `-dfexpr-after <simplify\|reaper>` | Dump fexpr after the given pass (#5153) |
| `-dfexpr-annot` | Dump fexpr (`prog.raw.fl`, `prog.simplify.fl`, `prog.reaper.fl`) alongside each `.cmx` (#5210) |
| `-dfexpr-annot-after <pass>` | As above but for a single pass; repeatable |

The existing `-dissector-partition-size` now validates `0 < size < 2 GiB`.

---

## Runtime & Memory

- **Faster minor-to-major promotion** (#5163) and GC prefetch tuning
  (#5416, #5359) — performance only, no API change.
- **`musl` compatibility** for runtime5 and `ocaml-jit` (#5123).

---

## Build System / Tooling

- Removed top-level `.depend` file (#5333).
- `configure` now checks required tool versions (upstream OCaml, menhir);
  accepts `--disable-tool-checks` (#5348).
- `configure` can set up an opam switch (#5360); ships a development OxCaml
  opam package pinning merlin + ocaml-lsp-server (#5344).
- Scrub environment when running tests (#5361); always install before tests
  (#5316).
- `:standard` added to `ocamlopt_flags` (#5209).
- `ocamlformat` upgraded to 0.28.1 (#5180, #5181) with tree-wide reformat.
- **Chamelon** test-case minimizer overhauled (subcommands, e.g.
  `chamelon run`; #5421 and ~10 related PRs).
- New `[%%expect_asm]` directive for expect tests (#5350): captures
  normalised architecture-specific assembly.
- Script to build individual files (#5404).

---

## Bug Fixes (user-visible)

- Backtraces no longer trashed by `Sys.getenv_opt` (#5076).
- Redundancy warning can be disabled on a single match pattern (#5202).
- `printtyp` "undefined modalities" crash on value descriptions (#5382);
  jkind error scoping (#5173); coerce error printing (#5457).
- Zero-alloc checker: `Widen` exception no longer escapes `V.join` (#5179).
- `%unsafe_ptr_set` with NULL base (#5087); symbol projections with
  unboxed numbers (#5299). Plus ~10 internal compiler fixes.

---

## Documentation Updates

- **New file**: `jane/doc/extensions/_06-kinds/implicit.md` (implicit kinds).
- **New content**: `jane/doc/extensions/_06-kinds/non-modal.md` (externality
  defaults, `_internal` suffix convention).
- `jane/doc/extensions/_03-unboxed-types/01-intro.md`: adds
  `value_or_null`, `void`, `bits8`, `bits16` to base-layout list; documents
  `#()` and `#false`/`#true`; mixed-block-layout macro bumped to v5.
- `jane/doc/extensions/_03-unboxed-types/03-block-indices.md`: new 52/12-bit
  offset/gap figures.
- `jane/doc/extensions/_05-modes/intro.md`: better explanation of
  mode-crossing.
- `jane/doc/extensions/_05-modes/syntax.md`: corrected tuple/mode examples.
- `jane/doc/extensions/_11-miscellaneous-extensions/zero_alloc_check.md`:
  updated error messages after `caml_make_vect` → `caml_array_make` rename.
- `HACKING.md`, `HACKING.ox.adoc`, `README.md` updated for 5.4.0.

---

## Breaking Changes and Upgrade Guide

These are changes that can break existing code without warning. Address them
in roughly the order listed; later ones depend on earlier ones being fixed.

### 1. New reserved keywords in the lexer

The lexer gained new reserved identifiers: `borrow_`, `poly_`, `repr_`,
`kind_`, plus `#false`, `#true`, and `#()` literals. Any existing code
using these names as plain identifiers will no longer parse.

**Fix**: Rename offending identifiers.

### 2. `kind_abbrev_` renamed to `kind_`

The `kind_abbrev_` keyword is gone. Parsetree constructors
`Psig_kind_abbrev` / `Pstr_kind_abbrev` are now `Psig_jkind` / `Pstr_jkind`
and carry a `jkind_declaration` record (`pjkind_name`, `pjkind_manifest`,
`pjkind_attributes`, `pjkind_loc`).

**Fix** in user code: `kind_abbrev_ foo = ...` → `kind_ foo = ...`.

**Fix** for `compiler-libs` consumers: update constructor names and record
fields. Note also that `pjkind_loc`/`pjkind_desc` on `jkind_annotation` are
now `pjka_loc`/`pjka_desc`, and `Pjk_abbreviation` now carries
`Longident.t loc` instead of `string`. `value_binding` gains `pvb_is_poly`;
`value_description` gains `pval_poly`.

### 3. Mixed block layout v4 → v5

`Stdlib_upstream_compatible.mixed_block_layout_v4` has been removed. C
sources using `Assert_mixed_block_layout_v4` must update the macro name
too. Block-index layout changed: 52-bit offset, 12-bit gap.

**Fix**: Rename references — these are mechanical.

```ocaml
(* Old *)
let _ = Stdlib_upstream_compatible.mixed_block_layout_v4

(* New *)
let _ = Stdlib_upstream_compatible.mixed_block_layout_v5
```

```c
/* Old */
Assert_mixed_block_layout_v4(block, layout);
/* New */
Assert_mixed_block_layout_v5(block, layout);
```

### 4. `Effect` low-level API refactor

The **public** `Effect.continuation` type is unchanged. The **internal**
`cont` GADT gained a termination type parameter (3-param now) and the
public `continuation` wraps it as `[@@unboxed]`. Code using only the
documented API (`Effect.Deep`/`Effect.Shallow` modules and their
`continuation`, `continue`, `discontinue`) keeps working.

Consumers of the **low-level runtime primitives** (`%resume`,
`%runstack`, `caml_alloc_stack`) must migrate:

- `caml_alloc_stack` + `%runstack` are merged into `%with_stack` (with
  `%with_stack_bind` as a dynamic-binding-aware variant).
- `%resume` now takes a continuation directly.

Bytecode gains a new RUNSTACK-family instruction.

### 5. `Obj.raw_field` / `Obj.set_raw_field` no longer `external`

They are now regular `val`s. Code that pattern-matches on externals or
rebinds through `%bytes` tricks needs to be reviewed.

### 6. Thread-local dynamic bindings removed

The runtime dynamic-variable API is reduced to `make`, `get`, and
`with_temporarily`. If you relied on per-thread root semantics, route
through `Basement.Dynamic` instead.

### 7. Boxed vectors now use tag 0 on amd64

Any code that reads vector block tags directly will read a different value.

### 8. Array runtime primitive bindings

`Array.make` and `Array.create_float` now bind to `caml_array_make` /
`caml_array_create_float` (matching upstream OCaml 5.4). Affects:

- **Zero-alloc checker expect tests** — error messages reference the new
  primitive names, so expected-output tests must be re-promoted.
- **Hand-written OCaml externals** that named `caml_make_vect` /
  `caml_make_float_vect` directly — rebind to the new names.

The old C entry points `caml_make_vect` / `caml_make_float_vect` are kept
as back-compat shims in `runtime/array.c` that forward to the new ones,
so external C code linking against these symbols continues to work.

### 9. Non-`value` base kinds now imply `mod external_`

`bits8`, `bits16`, `bits32`, `bits64`, `float32`, `float64`,
`untagged_immediate`, `vec128`, `vec256`, `vec512`, `void`, `word` all imply
externality. If you want the unmoded form, use the `_internal` suffix:

```ocaml
(* Before 5.2.0minus-31: bits64 did not imply mod external_ *)
(* After: these are the same *)
type t : bits64
type t : bits64_internal mod external_

(* If you specifically need the non-external form *)
type t : bits64_internal
```

### 10. Separability is part of the layouts extension

`separable` no longer requires its own extension flag. If your build enables
a standalone separability extension, drop that flag and rely on `layouts`.

### 11. Bootstrap compiler requirement bumped to OCaml 5.4.0

The **runtime OxCaml produces is still `5.2.0+ox`**, but building OxCaml
now requires an upstream OCaml 5.4.0 compiler on `PATH` (previously
`4.14.1`). Developers need to recreate their build switch. The
recommended flow is to install the `oxcaml-dev` opam package as
described in `HACKING.md` / `README.md`. As part of the bootstrap
work, OxCaml's own dune file now passes explicit `-I +unix` instead
of relying on auto-include, which silences
`ocaml_deprecated_auto_include` warnings during self-build (#5213).

Selected 5.4.0 stdlib features were **backported** rather than brought
in wholesale — notably `Format_doc` (new module surface) and the
`caml_array_make` runtime symbol rename. Code that relied on
identifiers renamed or removed in upstream 5.4.0 is only affected if
those changes were among the backports; consult the PR list in the
"Bootstrap Toolchain" section above.

### 12. `ocamlformat` 0.28.1

The tree has been reformatted with 0.28.1. Downstream projects may wish to
follow for diff-friendly merges.

### 13. Parseflambda rules

`make regen-flambda2-parser` is the way to regenerate fexpr parser message
rules after the #5221 primitive-descriptor rework.

### Upgrade checklist

A practical sequence for bumping:

0. **Update the bootstrap switch**: OxCaml builds now require upstream
   OCaml 5.4.0 on `PATH` (was 4.14.1). Follow the flow in `HACKING.md`
   (install `oxcaml-dev` as the switch invariant). The runtime OxCaml
   produces is still `5.2.0+ox` — this step only affects how you build
   OxCaml itself.
1. `git grep -l kind_abbrev_` → rename to `kind_`.
2. `git grep -l mixed_block_layout_v4` and `Assert_mixed_block_layout_v4` →
   bump to v5.
3. Only needed if you use low-level effect primitives: search for
   `%resume`, `%runstack`, `caml_alloc_stack` and migrate to
   `%with_stack` / `%with_stack_bind`. The public `('a,'b) continuation`
   API is unchanged, so users of `Effect.Deep` / `Effect.Shallow` with
   `continue` / `discontinue` need no changes.
4. Re-promote zero-alloc-checker expect tests (error messages now
   reference `caml_array_make` / `caml_array_create_float`). Rebind any
   hand-written OCaml externals that named the old symbols. C code
   linking against `caml_make_vect` / `caml_make_float_vect` continues
   to work via back-compat shims.
5. Check identifiers for collisions with `borrow_`, `poly_`, `repr_`,
   `kind_`.
6. If you relied on `bits*` being non-external, audit and switch to
   `bits*_internal`.
7. Audit any code using `Obj.raw_field` as an `external`.
8. If you used per-thread dynamic bindings in the low-level API, migrate
   to `Basement.Dynamic`.
9. Rebuild; watch for the new warning 216 (`use-during-borrowing`) if
   you adopt `borrow_`.
10. If you render jkinds in tooling, consider wiring `-kind-verbosity`.
