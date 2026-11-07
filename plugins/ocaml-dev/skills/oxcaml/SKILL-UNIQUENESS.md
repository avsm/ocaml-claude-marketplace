---
name: oxcaml-uniqueness
description: "OxCaml uniqueness types for tracking single-reference values enabling safe mutation and resource management"
---

# OxCaml Uniqueness: Detailed Guide

Uniqueness tracking enables safe in-place mutation and resource
management without runtime checks.

| Mode | Meaning | Subtyping | Crosses |
|------|---------|-----------|---------|
| `unique` | Exactly one reference to this value exists | ≤ `aliased` | Immediates, unboxed numbers, function types |
| `aliased` | Multiple references may exist | (top) | - |
| `many` | Closure can be called any number of times | ≤ `once` | Types containing no functions |
| `once` | Closure can be called at most once | (top) | - |

- Uniqueness (`unique`/`aliased`) and linearity (`once`/`many`) are
  related: a closure capturing a unique value can only be called once
  (otherwise it would use the unique value twice).
- `unique ≤ aliased`: a unique value can be used where aliased is
  expected (you can forget uniqueness); passing an aliased value where
  unique is expected is a type error.
- The legacy `unique_`/`once_` prefixes are removed, and `borrow_` has
  placement rules: see [SKILL-MODES.md](SKILL-MODES.md).

---

## Syntax

```ocaml
val allocate : unit -> bytes @ unique                  (* returns a unique value *)
val free : bytes @ unique -> unit                      (* consumes the only reference *)
val duplicate : 'a -> ('a * 'a) @ aliased
val register_callback : (unit -> unit) @ once -> unit  (* can only be invoked once *)
val set_handler : (int -> unit) @ many -> unit         (* can be invoked repeatedly *)
```

On parameters: `let free (x @ unique) = ...`,
`let copy (x @ aliased) = ...`. Modes of all axes combine:
`let f (x @ local unique once) = ...` (see also `with_ref` below).

---

## Mode Crossing

Some types cross uniqueness (can be treated as unique even when aliased):

```ocaml
(* Immediates cross uniqueness *)
let f (x @ aliased) : int @ unique = x  (* OK: int crosses *)

(* Function types cross uniqueness too *)
let h (f @ aliased) : (int -> int) @ unique = f  (* OK *)

(* Boxed data does NOT cross, even when immutable *)
let g (lst @ aliased) : int list @ unique = lst
(* ERROR: This value is aliased but is expected to be unique. *)
```

A type crosses uniqueness if:
- It's an immediate (int, bool, char, etc.) or an unboxed number (`float#`)
- It's a function type

Lists, records, refs and other boxed data do **not** cross uniqueness
— a unique list really must have a single reference. (Linearity is
different: any type containing no functions crosses `once`/`many`.)

---

## Aliased Modality on Fields

Store aliased values in unique containers:

```ocaml
(* Contents are aliased even if container is unique *)
type 'a aliased_box = { value : 'a @@ aliased } [@@unboxed]

(* Container is unique, contents are aliased *)
type 'a collection = {
  data : 'a aliased_box array;  (* elements aliased *)
}

(* Can free container while elements remain accessible elsewhere *)
val insert : 'a @ aliased -> 'a collection @ unique -> 'a collection @ unique
val free_container : 'a collection @ unique -> unit
```

```ocaml
type container = {
  handle : int ref;           (* unique with container *)
  cache : string @@ aliased;  (* aliased, shared elsewhere *)
}
let free (x @ unique) = ()

(* Can free handle; cache is not freed and may still be used elsewhere *)
let close_container (c @ unique) = free c.handle

(* Reading an aliased field returns an aliased value *)
let get_cache (c @ unique) : string (* aliased *) = c.cache
```

---

## Uniqueness Analysis

The compiler tracks unique values through plain rebindings and
per-field uses:

```ocaml
type sub
type t = { a : sub; b : sub }
let free (x @ unique) = ()

let ok (r @ unique) =
  free r.a;
  free r.b      (* OK: different fields *)

let ok2 (r @ unique) =
  let s = r in  (* s and r are aliases, both tracked *)
  free s.a;
  free r.b      (* OK: compiler knows s = r *)
```

- **Aliasing via binding**: in `let y = x in free x; y`, binding `y`
  alone is fine (a tracked alias of `x`) and `free x` consumes the
  value, but then using `y` is an error: "This value is used here, but
  it has already been used as unique at: ...".
- **Whole after part**: `free r.a; free r` is an error (`r.a` already
  freed, `r` contains it): "This value is used here, but part of it
  has already been used as unique at: ...".
- **Complex aliases are not tracked**: in
  `let s = Fun.id r in free s.a; free r.b`, `s` is the result of a
  call, so it is aliased and `free s.a` fails: "This value is aliased
  because it is the field a of the record at ... which is aliased.
  However, the highlighted expression is expected to be unique."

### Branches

Mutually exclusive branches (`if` or `match`) can use a unique value
differently:

```ocaml
let free (x @ unique) = ()
let store (x @ aliased) = ()

let ok (x @ unique) condition =
  if condition then
    free x              (* Use uniquely in this branch *)
  else
    (store x; store x)  (* Use as aliased in this branch *)

let ok2 (x : int option @ unique) =
  match x with
  | Some _ -> free x
  | None -> ignore x    (* aliased use *)
```

---

## Once Closures

Closures capturing unique values are `once`; a `once` closure may
consume what it captured, since it runs at most once:

```ocaml
let free (x @ unique) = ()

let make_callback (resource @ unique) : (unit -> unit) @ once =
  fun () -> free resource

let on_complete (f : (int -> unit) @ once) = f 1
let use_resource (r : int ref @ local) (x : int) = ()   (* borrows *)
let example (resource : int ref @ unique) =
  on_complete (fun result ->
    use_resource (borrow_ resource) result;
    free resource)  (* safe: callback runs at most once *)
```

Calling the resulting callback twice (`callback (); callback ()`) is
an error (see Common Errors). A `@@ many` field rejects `once`
closures:

```ocaml
type callback_data = {
  f : (unit -> unit) @@ many;  (* Always many *)
}

let store (cb @ once) : callback_data @ unique =
  { f = cb }  (* ERROR: cb is once, field requires many *)
```

---

## Patterns

### Resource Management

```ocaml
type handle

val open_file : string -> handle @ unique
val read : handle @ unique -> string * handle @ unique
val close : handle @ unique -> unit

let process_file path =
  let h = open_file path in
  let content, h = read h in
  close h;  (* Safe: h is unique *)
  content
```

### Unique Mutable Phase

```ocaml
type 'a builder

val create : unit -> 'a builder @ unique
val add : 'a -> 'a builder @ unique -> 'a builder @ unique
val freeze : 'a builder @ unique -> 'a array (* aliased, immutable *)

(* Thread the unique builder through explicit recursion: Stdlib's
   List.fold_left takes and returns its accumulator aliased *)
let build_array items =
  let rec go (b @ unique) = function
    | [] -> freeze b          (* Returns immutable array *)
    | x :: xs -> go (add x b) xs
  in
  go (create ()) items
```

### Threading Unique Values

```ocaml
(* BAD: can't use unique in loop directly *)
let bad (arr @ unique) =
  for i = 0 to 10 do
    set arr i 0  (* ERROR: arr used multiple times *)
  done

(* GOOD: thread unique through; borrow_ for the read-only use so it
   doesn't count as a (second) unique use.
   val length : arr @ local -> int *)
let rec set_all (arr @ unique) i =
  if i >= length (borrow_ arr) then arr
  else set_all (set arr i 0) (i + 1)

let good (arr @ unique) = set_all arr 0
```

### Borrowing Pattern

```ocaml
(* Temporarily use unique value, get it back *)
val with_ref : t @ unique -> (t @ local -> 'a) -> 'a * t @ unique

let example (data @ unique) =
  let result, data = with_ref data (fun d ->
    compute_something d
  ) in
  free data;
  result
```

---

## Common Errors and Solutions

- **"This value is used here, but it has already been used as
  unique"** — `let y = x in free x; y`: `free x` consumes `x`, so its
  alias `y` can't be used afterwards. Fix: don't alias before the
  unique use (`free x`), or use the alias instead
  (`let y = x in free y`).
- **"This value is used here as unique, but part of it has already
  been used"** — `match t with { field } -> use_field field; free t`
  (the error points at `use_field field`, not at the pattern). Fix:
  the `with_ref` pattern above — do the `match` and `use_field field`
  inside the callback, then `free` the `t` it returns.
- **"This value is once ... expected to be many"** —
  `let f () = free r in register_handler f` where the handler expects
  `many` and `r` is unique. Fix: don't capture a unique value in a
  `many` closure.
- **"This value is used here, but it is defined as once and has
  already been used"** — a `once` closure called twice:
  `let bad (f @ once) = f (); f ()`, or `let f () = free r in f (); f ()`
  with `r` unique. Fix: only call once (`let good (f @ once) = f ()`),
  or require many (`let good2 (f @ many) = f (); f ()`), or thread the
  unique value through instead of capturing it
  (`let r = use r in let r = use r in free r`).
- **"This value is aliased but is expected to be unique"** — an
  aliased value was used where unique is expected and its type does
  not cross uniqueness (see Mode Crossing).

---

A curried function whose `unique` first parameter is not itself
consumed prints as `a @ unique -> (b -> c)` (the partial application
is `many`); `a @ unique -> b -> c` means the partial application is
`once`. See [SKILL-MODES.md](SKILL-MODES.md).

See also: [SKILL-MODES.md](SKILL-MODES.md) for full mode system,
[SKILL-STACK-ALLOCATION.md](SKILL-STACK-ALLOCATION.md) for locality.
