---
name: oxcaml-zero-alloc
description: "OxCaml [@zero_alloc] compile-time verification that functions perform no heap allocations"
---

# OxCaml Zero-Alloc Checking: Detailed Guide

The `[@zero_alloc]` attribute enables compile-time verification that functions
don't allocate on the OCaml heap.

```ocaml
(* Simple zero-alloc function *)
let[@zero_alloc] add x y = x + y

(* With explicit type *)
let[@zero_alloc] multiply : int -> int -> int = fun x y -> x * y
```

## What Counts as Allocation?

**Forbidden** (causes check failure):
- Heap allocation: tuples, records, variants, closures, etc.
- Boxing: `int64`, `float` operations that box
- Indirect calls (function passed as argument)
- Under `strict` only: allocation on exception paths (the default
  relaxed semantics ignores allocations on a path that ends in a
  raise — **but only when compiling with `-g`**, see below)

**Allowed**:
- Stack allocation: `stack_`, `local_`
- Immediate values: `int`, `bool`, `char`
- Unboxed operations: `float#`, `int32_u`, `int64_u`, etc.
- Direct calls to known zero-alloc functions (including inlined
  `[@inline always]` helpers and `[@@noalloc]` externals, which are
  trusted)

```ocaml
let[@zero_alloc] with_pair x y =
  let local_ p = stack_ (x, y) in  (* OK: stack allocation *)
  fst p + snd p

let[@zero_alloc] sum_array (arr : int array) =
  let local_ count = ref 0 in  (* OK: local ref *)
  let mutable total = 0 in     (* OK: no allocation *)
  for i = 0 to Array.length arr - 1 do
    total <- total + arr.(i);
    incr count
  done;
  total + !count

let[@zero_alloc] unboxed_add (x : float#) (y : float#) : float# =
  Stdlib_upstream_compatible.Float_u.add x y  (* No boxing - zero-alloc *)

let[@inline always][@zero_alloc] helper x = x + 1

let[@zero_alloc] process (arr : int array) =
  for i = 0 to Array.length arr - 1 do
    arr.(i) <- helper arr.(i)  (* Inlined, so zero-alloc *)
  done

(* [@@noalloc] on externals is trusted *)
external fast_memcpy : bytes -> bytes -> int -> unit =
  "caml_fast_memcpy" [@@noalloc]

let[@zero_alloc] copy src dst len =
  fast_memcpy dst src len  (* OK: external marked noalloc *)
```

---

## Annotation Variants

| Attribute | Behaviour |
|-----------|-----------|
| `[@zero_alloc]` | Checked in all builds; "relaxed" semantics (allows allocation on exception paths) |
| `[@zero_alloc strict]` | No allocation on any path, including exceptions |
| `[@zero_alloc opt]` | Only checked in optimized builds (`-zero-alloc-check all`); useful when inlining is required for the function to be zero-alloc |
| `[@zero_alloc assume]` | Not checked — trusted to be zero-alloc; use sparingly. Also usable on one call site: `(f [@zero_alloc assume]) x` |
| `[@zero_alloc assume error]` | Marks the function as an error path — all code after a call to it is ignored |
| `[@zero_alloc assume_unless_opt]` | Assumed in debug builds, checked in optimized builds (`-zero-alloc-check all`) |
| `[@zero_alloc ignore]` | Exempts a function from `[@@@zero_alloc all]` |
| `[@zero_alloc arity n]` | In signatures: number of arguments before zero-alloc applies (see "In Signatures") |

```ocaml
let[@cold][@zero_alloc assume] log_error msg =
  Printf.eprintf "Error: %s\n" msg

let[@cold][@zero_alloc assume error] fatal msg =
  Printf.eprintf "Fatal: %s\n" msg;
  exit 1

let[@zero_alloc] process x =
  if x < 0 then log_error "bad input";  (* Allowed due to assume *)
  if x = 0 then fatal "unrecoverable";
  (* Code after fatal is ignored for zero_alloc purposes *)
  x + 1
```

### Relaxed Semantics: Exception Paths Need `-g`

```ocaml
let[@zero_alloc] find (arr : int array) idx =
  if idx < 0 || idx >= Array.length arr then
    raise (Invalid_argument "out of bounds")  (* Allowed - exception path *)
  else
    arr.(idx)
```

- The `(arr : int array)` annotation matters. With a polymorphic
  `'a array`, `arr.(idx)` may read a flat float array and box the
  result: `Error: allocation of 16 bytes for float`.
- **The exception-path relaxation needs `-g`.** Without `-g`, `raise`
  compiles to `raise_notrace` and is not recognised as an error path,
  so allocations feeding the exception are reported:

  ```ocaml
  let[@zero_alloc] check x =
    if x < 0 then raise (Invalid_argument (string_of_int x)) else x
  (* ocamlopt -c     : fails — "called function may allocate (external
                       call to caml_format_int)", "allocation of 24 bytes"
     ocamlopt -g -c  : passes
     [@zero_alloc strict] fails either way (with -g: "allocation of
     24 bytes on a path to exceptional return") *)
  ```

  (dune passes `-g` in both the dev and release profiles by default.)

---

## In Signatures

```ocaml
module M : sig
  val[@zero_alloc] fast_hash : string -> int

  (* arity: how many arguments before zero-alloc applies *)
  val[@zero_alloc arity 2] partial_apply : int -> int -> int

  type t = int -> int
  val[@zero_alloc arity 1] f : t  (* Need arity since t hides arrow *)
end = struct
  (* The implementation must be zero-alloc: checked even without an
     annotation here *)
  let fast_hash s = String.length s
  let partial_apply x y = x + y
  type t = int -> int
  let f x = x + 1
end

(* Only fully applied calls are zero-alloc *)
let f = M.partial_apply 1    (* May allocate closure *)
let x = M.partial_apply 1 2  (* Zero-alloc *)
```

### Implementing a `[@@zero_alloc]` val with an `external`

An `external` in the implementation can satisfy a
`val … [@@zero_alloc]` in the signature:

```ocaml
module M : sig
  val to_int : float -> int [@@zero_alloc]
  val len : string -> int [@@zero_alloc]
end = struct
  external to_int : float -> int = "caml_int_of_float" [@@noalloc]
  external len : string -> int = "%string_length"
end
```

The primitive is really checked:

- `%`-primitives are analysed. `external add : int32 -> int32 -> int32
  = "%int32_add"` against `val add : … [@@zero_alloc]` fails with
  `Error: allocation of 24 bytes for boxed_int32`.
- C stubs marked `[@@noalloc]` are trusted; an un-annotated C stub
  fails with `called function may allocate (external call to
  caml_add_float)`.
- The `external`'s arity must equal the `val`'s zero_alloc arity
  (`zero_alloc arity mismatch: … the syntactic arity of the
  implementation must match the function type in the interface`).
- A kind constraint in the signature can make a primitive pass:
  `val equal : ('a : immediate). 'a -> 'a -> bool [@@zero_alloc]`
  over `external equal : 'a -> 'a -> bool = "%equal"`.

---

## File-Level Annotations

```ocaml
[@@@zero_alloc all]

let add x y = x + y      (* Checked *)
let mul x y = x * y      (* Checked *)

let[@zero_alloc ignore] debug x =
  Printf.printf "%d\n" x  (* Exempted *)
```

The floating attribute accepts six payloads (anything else is warning
47 `attribute-payload`):

| Payload | Effect |
|---------|--------|
| `[@@@zero_alloc all]` | Annotate every function in the file (same as `-zero-alloc-assert all`) |
| `[@@@zero_alloc all_opt]` | Annotate every function as `[@zero_alloc opt]` (`-zero-alloc-assert all_opt`) |
| `[@@@zero_alloc check]` | Check non-`opt` annotations in this file (like `-zero-alloc-check default`) |
| `[@@@zero_alloc check_opt]` | Check only `opt` annotations (`-zero-alloc-check opt`) |
| `[@@@zero_alloc check_all]` | Check all annotations including `opt` (`-zero-alloc-check all`) |
| `[@@@zero_alloc check_none]` | Check nothing in this file (`-zero-alloc-check none`) |

---

## Understanding Failures

When the check fails, the compiler shows all allocations:

```
File "m.ml", line 2, characters 5-15:
2 | let[@zero_alloc] f x =
         ^^^^^^^^^^
Error: Annotation check for zero_alloc failed on function M.f (camlM__f_1_5_code).

File "m.ml", line 3, characters 8-13:
3 |   match bar x with
            ^^^^^
Error: called function may allocate (direct call camlM__bar_0_4_code)

File "m.ml", line 4, characters 14-20:
4 |   | a :: _ -> (a, a)
                  ^^^^^^
Error: allocation of 24 bytes
```

Message forms you will see: `allocation of N bytes` (sometimes with a
reason: `… for float`, `… for boxed_int32`), `called function may
allocate (direct call <symbol>)`, `(indirect call)`,
`(indirect tailcall)`, `(external call to <c_symbol>)`; under `strict`
each gains `on a path to exceptional return` where relevant. The
header reads `zero_alloc strict` for that variant (an `opt`
annotation is still reported as plain `zero_alloc`).

## Common Errors and Fixes

| Error | Example cause | Fix |
|-------|---------------|-----|
| `called function may allocate (indirect call)` — `(indirect tailcall)` when the call is in tail position | `let[@zero_alloc] apply f x = f x` | Can't fix without knowing `f` is zero-alloc: use `[@zero_alloc assume]` on the call, make `f` an inline function, or accept that this can't be zero-alloc |
| `allocation of N bytes` | tuple: `let[@zero_alloc] bad x y = (x, y)` | Stack-allocate (`exclave_ stack_ (x, y)`) or return unboxed (`#(x, y)`) |
| `allocation of N bytes` | record update: `{ s with x = s.x + 1 }` | Use mutable fields (`s.x <- s.x + 1; s`) or unboxed |
| `allocation of N bytes` | `Ok v` / `Error e` in a `result`-returning function | Return an `or_null` (or an unboxed tuple), or stack-allocate the result with `exclave_ stack_ (Ok …)` |
| `called function may allocate (external call to <c_symbol>)` | `external` without `[@@noalloc]` | Add `[@@noalloc]` if truly non-allocating, or assume at the call site |

```ocaml
let[@zero_alloc] good x y = exclave_ stack_ (x, y)
let[@zero_alloc] good2 (x : int) (y : int) : #(int * int) = #(x, y)

type state = { mutable x : int; mutable y : int; mutable z : int }
let[@zero_alloc] good_update s =
  s.x <- s.x + 1;
  s

external process : int -> int = "caml_process"
let[@zero_alloc] f x = (process [@zero_alloc assume]) x
```

`or_null` only accepts **value** types (`float# or_null` is a kind
error), but for value payloads it avoids the `Some` / `Ok` allocation:

```ocaml
let[@cold][@zero_alloc assume error] report_error code : char or_null =
  Printf.eprintf "Error %d\n" code;
  Null

let[@zero_alloc] parse_packet buf : char or_null =
  if Bytes.length buf < 4 then report_error 1
  else This (Bytes.get buf 0)
```

---

## Build Configuration

dune: `(library (name mylib) (ocamlopt_flags (:standard -zero-alloc-check all)))`

| Flag | Effect |
|------|--------|
| `-zero-alloc-check default` | Check non-opt annotations (default) |
| `-zero-alloc-check all` | Check all including opt |
| `-zero-alloc-check opt` | Check only opt annotations |
| `-zero-alloc-check none` | Disable checking of annotations |
| `-zero-alloc-assert {default\|all\|all_opt}` | Add annotations to all functions (`all` = `[@@@zero_alloc all]`, `all_opt` = `[@@@zero_alloc all_opt]`) |
| `-disable-zero-alloc-checker` | Assume everything may allocate; no function summaries computed (stronger than `-zero-alloc-check none`) |
| `-disable-precise-zero-alloc-checker` | Assume forward calls and mutually recursive functions may allocate (workaround when the precise analysis is too slow) |
| `-zero-alloc-checker-details-cutoff <n>` | Max error locations shown per failing function (default 20; negative, e.g. `-1` = all; `0` = none, just fail) |
| `-zero-alloc-checker-details-extra` / `-no-…` | Show extra detail in checker errors |
| `-zero-alloc-checker-join <n>` | Abstract paths kept before widening (default 100; negative = fail instead of widening; 0 = keep all) |

---

## Limitations and Tips

- Conservative - may reject valid zero-alloc code
- Higher-order functions often fail (indirect calls)
- Requires cross-module info for accurate checking — annotate
  signatures
- Flat float optimization can cause false positives (polymorphic
  `arr.(i)`; annotate the element type)
- Safepoints not considered allocating (may context switch)
- Start with `[@zero_alloc opt]` until code stabilizes
- Use `[@cold]` on error paths to hint to the optimizer

See also: [SKILL-STACK-ALLOCATION.md](SKILL-STACK-ALLOCATION.md) for local allocation,
[SKILL-UNBOXED.md](SKILL-UNBOXED.md) for unboxed types.
