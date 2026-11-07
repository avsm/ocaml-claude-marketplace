# OxCaml Changes: 5.2.0minus-38 to 5.2.0minus-39

This document summarizes the key changes between OxCaml versions
5.2.0minus-38 (2026-05-05) and 5.2.0minus-39 (2026-05-22). The compiler
ships as `ocaml-variants.5.2.0+ox`, on the OCaml 5.2 base. For what
came next (`5.2.0minus-40`, the OCaml 5.4 rebase, `5.4.0-ox1` …
`5.4.0-ox9`) see [CHANGES-39-ox9.md](CHANGES-39-ox9.md). Note that
several minus-39 rules described here were relaxed or reversed later
(`[@@flatten_floats]` became optional; `int64#`-style names were
removed at ox7).

## At a glance

For users tracking the headline changes:

- **`unique_` and `once_` prefix syntax removed** (#5931). Use
  `(x @ unique)` / `(e : @ unique)` mode syntax instead. `local_` is
  kept.
- **Labeled tuples are now always-on** (#5692). The `labeled_tuples`
  extension was deleted — `-extension labeled_tuples` is no longer a
  recognized flag. Repeated labels in one tuple are no longer allowed.
- **Float record representation overhaul** (#5795, #5796, #5774).
  Mixed `float`+`float#` records now **require `[@@flatten_floats]`**
  (a hard error without it); such records get no unboxed (`t#`)
  version. All-`float64` records are now mixed blocks by default;
  `[@@represent_as_float_array]` restores the old float-array layout.
  Mixed-block layout bumped to **v6**.
- **ikinds kind checker is now on by default** (#5999); disable with
  `-no-ikinds` (the old `-ikinds` flag is gone).
- **Block indices no longer support float records** (#6095).
- **Scannable axes now meet instead of override** (#5921) — a written
  scannable axis can only lower, never raise.
- **`Stdlib_stable.Iarray` lost its OxCaml-specific API** (#5693):
  `( .:() )` and every `*_local` function were removed, aligning with
  upstream 5.4's iarray ahead of a future merge.
- **New warning 182** `untagged-external-small-int-return`, on by
  default (#6081).
- **Preemption groundwork** (#4881): an `Effect.Preemption` effect
  constructor and `Domain.Tick.acquire`/`release` exist, but there is
  no way to run preemptible code yet (the handler API arrives in
  minus-40).
- **Merlin is now vendored** in `external/merlin/` and retagged in
  lockstep with each compiler release (#5568).
- **Magic number bumped** 577 → 578 (#6107): rebuild everything.

---

## Major Language Features

### `unique_` / `once_` prefix syntax removed (#5931)

**BREAKING.** The legacy keyword prefixes `unique_` and `once_` were
removed from the lexer and parser. `local_` is explicitly kept. Since
they are no longer keywords, old code now parses as an application of
an unbound identifier rather than giving a nice error.

```ocaml
(* Old *)
let dup (unique_ x) = (x, x)
let mk () = unique_ (1, 2)
val f : unique_ string -> once_ (unit -> unit)

(* New *)
let dup (x @ unique) = (x, x)
let mk () = ((1, 2) : @ unique)      (* expression: type-with-mode constraint *)
val f : string @ unique -> (unit -> unit) @ once
```

Note `(e @ unique)` is *pattern* syntax; in expression position use the
`(e : @ modes)` constraint form (available since minus-38).

### Labeled tuples always-on; extension removed (#5692)

Preparing for the future upstream 5.4 merge, OxCaml adopted upstream's
version of labeled tuples:

- `type t = x:int * y:string`, `~x:5, ~y:"hi"`, `let f (~x, ~y) = ...`
  all work **without any extension flag**.
- The `labeled_tuples` extension was deleted from the extension
  registry, so `-extension labeled_tuples` is now an **error**. Remove
  it from build flags.
- **Repeated labels in a single tuple are no longer allowed**
  (previously OxCaml accepted e.g. `~a:"s", 10, ~a:"hi"`).

### Float / `float64` record representations (#5795, #5796, #5774)

The representation of records involving `float` and `float#` changed.
State at minus-39:

- **Mixed `float`+`float#` records require `[@@flatten_floats]`.**
  Declaring such a record without the attribute is a hard error
  (`Missing_flatten_floats`); putting the attribute on a record that
  is not mixed `float`+`float64` is also an error
  (`Misplaced_flatten_floats`). With the attribute, the boxed `float`
  fields are stored flat (unboxed) inline — as minus-38 did
  automatically:

  ```ocaml
  type t = { f : float; u : float# }
  (* Error: missing [@@flatten_floats] *)

  type t_flat = { f : float; u : float# } [@@flatten_floats]
  (* ok; f stored flat inline *)
  ```

  `[@@flatten_floats]` records get no unboxed (`t#`) version, and the
  attribute is part of the representation for module inclusion checks.

- **All-`float64` records** (every field `float#`) are now represented
  as **mixed blocks** by default, instead of flat float-array blocks.
  `[@@represent_as_float_array]` opts back into the float-array
  representation; it errors on records that are not all-`float64`.

- The C-visible mixed-block layout version is now **v6**:
  `Stdlib_upstream_compatible.mixed_block_layout_v5` was renamed to
  `mixed_block_layout_v6`, and C stubs using
  `Assert_mixed_block_layout_v5` must move to v6. v6 also makes
  all-`value`/`void` mixed records and variants uniform blocks (#5774).

If you have C stubs or `Obj`-level code touching these records, audit
them; pure OCaml code without representation assumptions is unaffected
apart from the now-mandatory attribute.

### Scannable axes: meet instead of override (#5921)

**Semantics change.** Writing a scannable axis after a layout (e.g.
`value maybe_separable`) now takes the **meet** with the layout's
existing bound — it can only *lower* the axis, never raise it.
Previously it *overwrote* the axis. Consequences:

- `value maybe_separable` now equals `value` (since `value` already
  implies `separable`, and meet can't loosen it).
- `k mod axis` and `k axis` are now equivalent for scannable axes
  (`mod` spelling for scannable axes is deprecated).
- `value` is now defined as `value_or_null non_null separable`.

Kind annotations that relied on raising an axis change meaning or
error; expect tests quoting printed kinds may need re-promotion.

### Block indices: float records unsupported (#6095)

**BREAKING** for block-index users. Indices can no longer be taken into
float records (records where boxed `float`s are stored flat), including
via unboxed-record paths. Previously `(.f)` on `type t = { f : float }`
produced a `(t, float#) idx_imm` with a magic element type; now:
`Error: Block indices do not support float records.` The docs now list
the exclusions: `[@@unboxed]` records, `[@@represent_as_float_array]`
records, records storing `float`s flat, and `private` records.

### ikinds enabled by default (#5999); implicit kind redeclaration (#5980)

The "ikinds" kind-checking engine is now the default
(`-no-ikinds` opts out; the old opt-in `-ikinds` flag was **removed**).
This is intended to be behavior-preserving — an internal engine swap —
though some error-message wording differs.

Separately, re-declaring an implicit kind in the same signature is now
allowed; later declarations shadow earlier ones for subsequent items:

```ocaml
module type S = sig
  [@@@implicit_kind: ('a : immediate)]
  val before : 'a -> 'a          (* ('a : immediate) *)
  [@@@implicit_kind: ('a : bits64)]
  val after : 'a -> 'a           (* ('a : bits64) — previously an error *)
end
```

### Metaprogramming / quotations

All under `-extension-universe beta` with `runtime_metaprogramming`:

- **`open` inside quotes** (#6092): `let open M in ...`, `M.(...)`,
  `M.{...}`, `M.(Constructor ...)` now work inside `<[ ... ]>`,
  including opening first-class-module bindings and cross-stage opens
  with splices. Compiled as a shadowing `open!`, so generated code may
  trigger warnings 44/45. `let open struct ... end in ...` in a quote
  is rejected.
- **Current-unit references in quotes** (#6086): quotes may reference
  identifiers of the current compilation unit (top-level ids defined
  earlier in the same file). Caveat: `Eval.eval` of such a quote before
  the unit finishes initializing reads uninitialized values.
- **`[@magic_staged_modes]`** (#6066): expression attribute on a quote
  that delays mode checks until program generation.
  `<[ stack_ (Some 42) ]>` is normally rejected (a quoted expression's
  result must be at the legacy modes);
  `(<[ stack_ (Some 42) ]> [@magic_staged_modes])` is accepted and any
  violation surfaces at generation/eval time.
- **File-level `@@ static` modality** on an `.mli` makes the unit
  static when imported (#6041), affecting staging checks across units.
- Fixes: infix-operator printing/handling in quotes (#5982),
  constructor lambda generation (#5983), inclusion-error reporting for
  staged code (#6070), aliased interfaces may be absent when building
  the metaprogramming cmi_bundle (#6043).

### Smaller language-level items

- **Doc comments on `kind_` declarations** now attach correctly in
  signatures (#5987).
- **Warning 182 `untagged-external-small-int-return`** (#6081), on by
  default: fires when an `external` *returns* `(int8[@untagged])` or
  `(int16[@untagged])` — `[@untagged]` does no sign-extension; use
  `[@unboxed]` instead.
- **Existential kind annotations in GADT patterns** checked correctly
  (#5935).
- **Toplevel** prints layout-polymorphic values as `<lpoly>` (#5866).
- Soundness fix: `[@@unboxed]` + existentials (#6046).
- Clearer errors for invalid `[@unboxed]`/`[@untagged]` in `external`
  declarations (#5985).

---

## Runtime / Parallelism

### Preemption groundwork (#4881)

The initial implementation of fiber preemption landed, but at minus-39
it is **groundwork only — there is no way to run preemptible code**:

- `Stdlib.Effect` declares a `Preemption : unit t` effect constructor
  (explicitly marked unfinished in the source).
- `Stdlib.Domain.Tick` provides `acquire ~interval_usec` / `release`:
  between the two calls the runtime tick thread ticks at least as
  frequently as requested.

The usable API — `Effect.Deep.Preemptible` / `Shallow.Preemptible`
handlers, `tick_outcome`, `Domain.Tick.with_` — arrives in minus-40
(see CHANGES-39-ox9.md). Follow-up in this window: an
Onload-compatibility workaround for the tick thread (#6016).

### GC

- **Automatic compaction is skipped when the heap is a single chunk**
  (#6089) — compaction couldn't return memory to the OS anyway.
  Explicit `Gc.compact` is unaffected.

---

## Stdlib

- **`Stdlib_stable.Iarray` — BREAKING (#5693).** Aligning with
  upstream 5.4's iarray ahead of a future merge, ~53 OxCaml-specific
  exports were deleted: the `( .:() )` indexing operator and every
  `*_local` variant (`init_local`, `map_local`, `fold_left_local`,
  `to_seq_local`, `sort_local`, …). Migrate e.g. `a.:(n)` →
  `Iarray.get a n`. (`Stdlib_stable.IarrayLabels` retains both.)
- **`Random.State` functions now take the state `@ local`** (#5908):
  `copy`, `bits`, `int`, `full_int`, `int_in_range`, `int32`, `int64`,
  `nativeint`, the `_in_range` variants, `float`, `bool`, `bits32`,
  `bits64`, `nativebits`, `split`. Strictly more permissive.

---

## Compiler Flags

New flags (in `driver/oxcaml_args.ml` / `driver/main_args.ml`):

| Flag | Purpose |
|------|---------|
| `-no-ikinds` | Disable the (now default) ikinds-based kind checker. (#5999) |
| `-name-mangling-scheme {flat\|structured}` | Override the configure-time name-mangling scheme. Also an `OCAMLPARAM` key. (#5963) |
| `-flambda2-inline-small-functor-size <n>` / `-flambda2-inline-large-functor-size <n>` | Functor inlining size limits; large-functor limit effectively disabled by default (#5941). (#5878) |
| `-flambda2-speculative-inlining-track-lifted-constants` / `-no-...` | Count lifted constants during speculative inlining (default off). (#5917) |
| `-X <name>=<value>` | Generic registry for temporary internal knobs. Undocumented, unstable; `-X` is now reserved. (#6003) |

Removed flags (**breaking for build scripts**):

- `-ikinds` (now the default; only `-no-ikinds` remains).
- `-extension labeled_tuples` (extension deleted; see above).

New `OCAMLPARAM` keys: `gdwarf-inlined-frames` (#6061),
`name-mangling-scheme` (#5963), `X<name>=<value>` (#6003).

`configure.ac` is unchanged in this window — no new configure options.

---

## Tooling

- **Merlin vendored into the repo** (#5568): the full merlin source
  tree now lives at `external/merlin/` and is retagged in lockstep with
  each compiler release. It is *not* built or installed by the
  compiler's `make install` — it remains its own dune project
  (`merlin-lib`, `dot-merlin-reader`, `merlin`, `ocaml-index`), but
  sources are now versioned with the compiler.
- **LLDB**: OxCaml's DWARF tooling switched to a dedicated OxCaml LLDB
  language plugin (#4961), pinned at `21.1.0+oxcaml0` (#5996). Plain
  `b Module.fn` breakpoints work; inlined functor frames display as
  `[inlined]`; values print in OCaml syntax with mode annotations.
- **DWARF for unboxed-element arrays** (#6102): LLDB can now display
  `float# array`, `int64# array`, unboxed-product arrays, etc. (This
  is debug-info only — no new array types were added.)

---

## Breaking changes and upgrade guide

Address in roughly this order.

### 1. Recompile from clean

Magic number bumped 577 → 578 (#6107). All
`.cmi`/`.cmo`/`.cmx`/`.cma`/`.cmxa` from minus-38 are incompatible.

### 2. Replace `unique_` / `once_` prefixes

```ocaml
(* Old *)              (* New *)
let f (unique_ x) ...  let f (x @ unique) ...
unique_ (a, b)         ((a, b) : @ unique)
once_ ty               ty @ once
```

`local_` still works.

### 3. Drop `-extension labeled_tuples`

The flag now errors. Labeled tuples work unconditionally. If you used
repeated labels in one tuple, rename them — that feature is gone.

### 4. Audit float-record representations

- Mixed `float`+`float#` records now **require `[@@flatten_floats]`**
  — add it (such records lose their unboxed `t#` version).
- All-`float64` records became mixed blocks; add
  `[@@represent_as_float_array]` to restore the float-array layout.
- C stubs: `Assert_mixed_block_layout_v5` → `v6`;
  `Stdlib_upstream_compatible.mixed_block_layout_v5` →
  `mixed_block_layout_v6`.

### 5. Migrate `Stdlib_stable.Iarray` local API

`( .:() )` and all `*_local` functions are gone. Use `Iarray.get` and
the plain variants (or `IarrayLabels`, which retains them).

### 6. Block indices into float records

No longer expressible; restructure to avoid taking `idx` values into
records that store floats flat.

### 7. Kind annotations with scannable axes

Axes now meet rather than override — an annotation can only *lower* an
axis. Grep kind annotations for spellings that tried to *raise* one
(e.g. `value maybe_separable`, `immediate separable`): they are now
no-ops equal to the bare layout, so delete them or rethink the type.
Re-promote expect tests quoting printed kinds.

### 8. Build scripts: flag changes

`-ikinds` removed (default-on; escape hatch `-no-ikinds`).

### 9. `-warn-error` users: new warning 182

`external` declarations returning `(int8[@untagged])` /
`(int16[@untagged])` now warn. Switch to `[@unboxed]`.

### Upgrade checklist

1. **Recompile from clean** (magic 577 → 578).
2. **`unique_`/`once_` → `@ unique`/`@ once`.**
3. **Remove `-extension labeled_tuples`** from build flags; de-dup
   repeated tuple labels.
4. **Add `[@@flatten_floats]`** to records mixing `float` and `float#`
   (now mandatory); audit all-`float64` records
   (`[@@represent_as_float_array]` restores the old layout); bump C
   layout asserts v5 → v6.
5. **Migrate `Stdlib_stable.Iarray`** `*_local` / `.:( )` uses.
6. **Fix block-index uses** into float records.
7. **Re-check kind annotations** using scannable axes; re-promote
   kind-printing expect tests.
8. **Update build scripts** for the removed `-ikinds` flag.
9. **If using C stubs with small ints**: heed warning 182; prefer
   `[@unboxed]` returns.

---

## Compiler internals

Condensed; most users can stop reading here.

- **Flambda 2**: switch-table optimisation extended to naked numbers
  and symbols (#3242); functor inlining size limits (#5878, #5941);
  speculative-inlining lifted-constant sizing (#5917);
  continuation-specialization inlining replay fix (#5954);
  `imported_names` removal saga (#5886/#6085/#6100); Patricia-tree
  efficiency work (#6068, #5833, #5899); cmx-export efficiency
  (#5929); cmx-loading profiling (#5910).
- **reaper**: set-of-closures rewrite fixes (#6005), kind-rewrite fix
  (#5934), `if`-cascade reduction (#5938), code-size computation
  (#5793).
- **Backend/x86**: binary-emitter encodings — shorter 16-bit immediate
  arithmetic (#6027), NOP fix (#6028), nop-alignment in `.text`
  (#6029), canonical reg-reg moves (#6023), 2-byte VEX (#6012, #6033,
  #6022); improved int-comparison codegen (#5738); `c[lt]z`
  nonzero-arg intrinsics removed (#5827); regalloc dummy-use removal
  (#5614), affinity work (#5849, #5844).
- **Backend/arm64**: emitter ordering fix (#5978).
- **CFG**: `Printcfg` module (#5632), `Cfg_quick_hash` (#5788).
- **Types/modes internals**: total structural order for morph map keys
  (#6047), diamond-aware hints (#5858), `Record_boxed` sort removal
  (#6002), lambda GADT-match fix (#5879), negative sort-variable IDs
  in `.cmi` (#5663).
- **DWARF**: `DW_AT_name`/`DW_AT_linkage_name` flipped to match DWARF5
  (#5976); function sections; `-gdwarf-inlined-frames` via OCAMLPARAM.
- **fexpr** (Flambda 2 textual IR): syntactic sugar (#5474), structured
  primitive (#5825), stub info (#6004), syntax coloring (#5925).
