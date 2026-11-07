---
name: oxcaml-stack-allocation
description: "OxCaml stack allocation patterns using local_, stack_, and exclave_ for GC-free value allocation"
---

# OxCaml Stack Allocation: Detailed Guide

Stack-allocated values avoid GC overhead but cannot escape their
defining scope.

- **Global** values live on the heap and can be stored anywhere,
  returned from functions, and live indefinitely.
- **Local** values live on the stack and must not escape their
  defining scope.

## The Region Model

Local allocations do **not** go on the OS call stack: they live on a
separate, dynamically-grown local stack laid out like the minor heap.
Each function call opens a *region* on it; local values
(`local_`/`stack_`) are allocated in the current region and freed when
it ends. After `exclave_`, allocations go in the *caller's* region.

Because the local stack grows dynamically (it is not the ~8MB OS
thread stack), large local allocations won't crash — but they hold
memory for the whole region and forgo the minor heap's locality
benefits. Prefer the heap for very large or long-lived data (dubious:
a million `stack_` cons cells built via `make_list` below).

---

## Syntax

### Creating Local Values

```ocaml
let f () =                 (* a top-level binding cannot be local *)
  let local_ x = (1, 2, 3) in   (* modes the binding local: inference
                                   may stack-allocate *)
  let y = stack_ (1, 2, 3) in   (* only stack_ forces stack allocation *)
  (* With a type annotation — no parentheses around the pattern
     (`let local_ (z : int * int) = …` is a syntax error) *)
  let local_ z : int * int = (1, 2) in
  let (a, _, _) = x and (b, _, _) = y and (c, _) = z in
  a + b + c
```

### Local Parameters

```ocaml
let process (data @ local) =
  let mutable sum = 0 in
  for i = 0 to Array.length data - 1 do
    sum <- sum + data.(i)
  done;
  sum

let _ = process [| 1; 2 |]   (* a global argument is accepted too (subtyping) *)
```

A `@ local` parameter may be stack-allocated: it cannot be stored
globally, and it cannot be passed to functions that don't accept local
arguments. Stdlib's `Array.fold_left`, `List.iter`, `List.length`, …
take global values; write a loop or a local-accepting helper (e.g.
`sum_local` below), or use Base's local-aware functions.

A local *argument* may not be passed to a tail call (the region would
be closed before the callee runs): write `f local_arg [@nontail]`.

---

## Returning Local Values: `exclave_`

`exclave_`:
1. Ends the current function's region early
2. Evaluates its body in the *caller's* region, so allocations made
   there (`stack_ …`) can be returned
3. Must be at tail position

```ocaml
let make_pair x y = exclave_ stack_ (x, y)

(* Calling another local-returning function *)
let g x = exclave_ make_pair x x

(* if / match: all branches must be exclave-compatible *)
let h x = exclave_
  if x > 0 then stack_ (x, 1)
  else stack_ (0, 0)

(* OK: everything after exclave_ — including this stack_ allocation —
   happens in the CALLER's region and may be returned *)
let fine x = exclave_
  let local_ temp = stack_ (x, x) in
  temp

(* Caller receives a local value *)
let use () =
  let local_ p = make_pair 1 2 in
  fst p + snd p
```

Use this for e.g. temporary pairs for multiple returns
(`let divmod a b = exclave_ stack_ (a / b, a mod b)`).

### Local-Returning Tail Calls Need `exclave_`

Calling a local-returning function in tail position **requires**
`exclave_`, even when the result mode-crosses (e.g. is an `int`). The
usual trap is `fst`/`snd` (`[@local_opt]` primitives) on a local
tuple:

```ocaml
let g (p : (int * int) @ local) = fst p
(* Error: This local-returning application is in a tail position that
          is not enclosed in an exclave_ expression. *)

(* Fixes — all return a plain global int except the last *)
let g1 (p : (int * int) @ local) = let (a, _) = p in a
let g2 (p : (int * int) @ local) = let r = fst p in r
let g3 (p : (int * int) @ local) = fst p [@nontail]
let g4 (p : (int * int) @ local) = exclave_ fst p   (* int @ local *)
```

Not affected: `!r`, field projection or pattern matching on a local
value, `fst p + snd p` (the tail call is `+`).

---

## Mutable State: Refs, `let mutable`, Loops

References can be stack-allocated for mutable accumulators:

```ocaml
let sum_array arr =
  let local_ total = ref 0 in
  for i = 0 to Array.length arr - 1 do
    total := !total + arr.(i)
  done;
  !total  (* Return the int, not the ref *)
```

- **Refs (and mutable fields, and arrays) may themselves be local, but
  their *contents* must be `global`** — `acc := stack_ (...)` is a
  mode error, even if the ref is local.
- `let mutable` variables *can* hold local values, but only values
  from the variable's own region. **A `for`/`while` body is its own
  region**, so `acc <- stack_ (…)` inside a loop body is rejected. In
  loops, track state in scalars (or one unboxed pair:
  `let mutable best = #(0, arr.(0))`); a `stack_` tuple assigned
  inside the loop body would not outlive the body's region.
- To accumulate local data across iterations, use recursion with a
  local accumulator:

```ocaml
let rec sum_local acc (l @ local) =
  match l with
  | [] -> acc
  | x :: xs -> sum_local (acc + x) xs

let collect_positives arr =
  let rec go i (acc @ local) =
    if i = Array.length arr then sum_local 0 acc
    else if arr.(i) > 0 then
      go (i + 1) (stack_ (arr.(i) :: acc)) [@nontail]
    else go (i + 1) acc
  in
  go 0 [] [@nontail]

(* let mutable holding a local value (outside any loop body) *)
let no_loop () =
  let mutable acc = [] in
  acc <- stack_ (1 :: acc);
  sum_local 0 acc [@nontail]
```

---

## Local Data Structures

```ocaml
(* Lists: every level is exclave_, so each cons cell is allocated in
   the original caller's region.
   val make_list : int -> int list @ local -> int list @ local *)
let rec make_list n (acc @ local) = exclave_
  if n = 0 then acc
  else make_list (n - 1) (stack_ (n :: acc))
```

```ocaml
(* Stack-allocated array: stack_ on an array literal. (Array.make
   always heap-allocates; `let local_ a = Array.make …` only restricts
   the mode.) *)
let with_temp_array f =
  let arr = stack_ [| 0; 0; 0; 0 |] in
  f arr [@nontail]
  (* arr deallocated when function returns; [@nontail] is required
     because a local value can't be passed to a tail call *)
```

```ocaml
(* Records (here with unboxed fields - very efficient) *)
type local_point = { x : float#; y : float# }

let make_point x y = exclave_
  stack_ { x = Stdlib_upstream_compatible.Float_u.of_float x;
           y = Stdlib_upstream_compatible.Float_u.of_float y }
```

Closures capturing local data are themselves local:

```ocaml
let with_counter f =
  let local_ count = ref 0 in
  let local_ incr () = count := !count + 1 in
  f incr;
  !count
```

---

## Errors and Fixes

- **Returning a local** (`let bad () = let local_ x = (1, 2) in x`):
  "This value is local but is expected to be local to the parent
  region or global because it is a function return value." Fix:
  `let good () = exclave_ stack_ (1, 2)`, or allocate globally (the
  default: drop `local_`).
- **Storing a local in a global structure**
  (`let local_ x = (1, 2) in global_list := x :: !global_list`):
  "This value is local but is expected to be global". Fix: allocate
  `x` globally (drop `local_`), copy to the heap, or restructure.
- **`exclave_` not at tail**
  (`let y = exclave_ stack_ (x, x) in fst y`): "Exclave expression
  should only be in tail position of the current region." Fix: make
  `exclave_` the tail of the function (`let good x = exclave_
  stack_ (x, x)`) and destructure in the caller with
  `let (a, _) = p in a` — not `fst p` in tail position (see the
  tail-call rule above).
- **Returning a closure that captures a local**
  (`let local_ x = ref 0 in let f () = !x in f`): "This value is local
  because it closes over the value x …". Fix: use `let mutable`
  (mutable variables don't create closures; e.g.
  `let mutable x = 0 in for i = 1 to 10 do x <- x + i done; x`) or
  ensure the closure is also local.
- **Storing a local value in a ref or mutable field**
  (`let local_ acc = ref [] in acc := stack_ (1 :: !acc)`): "This
  value is local because it is stack_-allocated. However, the
  highlighted expression is expected to be global." Fix: `let mutable`
  (see `no_loop` above).
- **Assigning a `stack_` value to a `let mutable` inside a loop
  body**: "local because it is stack_-allocated … expected to be local
  to the parent region". Fix: recursion with a local accumulator.
- **Top-level `let local_ x = …`**: "The expression is local but is
  expected to be global because it is the value x in the structure
  …". Local bindings must be inside a function body.

---

## When to Use

- **Use for**: short-lived intermediate values, accumulators in loops,
  temporary pairs/tuples for multiple returns, processing pipelines
  where data flows through.
- **Avoid for**: values stored in long-lived data structures, values
  returned from deeply nested call chains, very large allocations
  (held for the whole region), values shared across threads.

## With Other Features

- **Unboxed types**: see `make_point` above; details in
  [SKILL-UNBOXED.md](SKILL-UNBOXED.md).
- **Zero-alloc**: stack allocation satisfies `zero_alloc`:

```ocaml
let[@zero_alloc] process x y =
  let local_ p = stack_ (x, y) in
  fst p + snd p
```

- **Comprehensions**: list comprehensions build their *intermediate*
  structures on the local stack for efficiency, but the resulting list
  is always **global** (heap-allocated) — you cannot get a local list
  out of a comprehension.

See also: [SKILL-MODES.md](SKILL-MODES.md) for the full mode system,
[SKILL-UNBOXED.md](SKILL-UNBOXED.md) for unboxed types.
