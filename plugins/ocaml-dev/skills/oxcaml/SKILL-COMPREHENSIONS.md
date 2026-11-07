---
name: oxcaml-comprehensions
description: "OxCaml list and array comprehension syntax for concise collection building with filters and nested iteration"
---

# OxCaml Comprehensions: Detailed Guide

Comprehensions build lists, arrays and immutable arrays.

The extension is **Beta** maturity and **not enabled by default**:
pass `-extension comprehensions` (or `-extension-universe beta`), e.g.
via dune `(flags (:standard -extension comprehensions))`. Without it:

```
Error: The extension "comprehensions" is disabled and cannot be used
```

## Syntax

```ocaml
(* Lists: ranges, filter, iteration over a list, downto *)
let l1 = [ x * 2 for x = 1 to 10 ]
(* [2; 4; 6; 8; 10; 12; 14; 16; 18; 20] *)
let l2 = [ x for x = 1 to 20 when x mod 3 = 0 ]
(* [3; 6; 9; 12; 15; 18] *)
let l3 = [ String.uppercase_ascii s for s in ["hello"; "world"] ]
(* ["HELLO"; "WORLD"] *)
let l4 = [ x for x = 10 downto 1 ]
(* [10; 9; 8; 7; 6; 5; 4; 3; 2; 1] *)

(* Arrays: same syntax with [| |] *)
let a1 = [| x * x for x = 1 to 5 |]              (* [|1; 4; 9; 16; 25|] *)
let a2 = [| x + 1 for x in a1 |]                 (* from an array *)

(* Immutable arrays: [: :] *)
let i1 = [: x for x = 1 to 10 when x mod 2 = 0 :]   (* int iarray *)

(* Patterns in iterators *)
let l5 = [ a + b for (a, b) in [(1,2); (3,4); (5,6)] ]   (* [3; 7; 11] *)
```

- **Ranges** are empty if start > stop (`to`) or start < stop
  (`downto`): `[ x for x = 5 to 3 ]` is `[]`.
- The source of a `for … in` iterator must be the **same collection
  kind as the comprehension**: lists in `[ … ]`, arrays in `[| … |]`,
  iarrays in `[: … :]`. Mixing them is a type error ("The value a has
  type int array but an expression was expected of type 'a list because
  it is in a for-in iterator in a list comprehension") — convert first
  (`Array.to_list`, `Iarray.to_list`, …).
- The source must also be **global**: iterating over a `local`
  collection is rejected ("This value is local to the parent region but
  is expected to be global").
- **Unboxed types are banned**: the body must have layout `value`, as
  must the element type of any array you iterate over. So
  `[| Float_u.of_float x for x in ... |]` is a type error — build
  `float# array` / `int8# array` values with explicit loops instead.
- **Partial patterns may raise**: `[ x for Some x in options ]` raises
  if a `None` is encountered. Filter explicitly instead (see
  `filter_map` and `somes` under "Common Patterns").

---

## Nested vs Simultaneous Iteration

```ocaml
(* Nested: for ... for ... *)
let n1 = [ (x, y) for x = 1 to 3 for y = 1 to 2 ]
(* [(1,1); (1,2); (2,1); (2,2); (3,1); (3,2)]   3 * 2 = 6 elements *)
let n2 = [ (x, y) for x = 1 to 3 for y = 1 to x ]  (* inner uses outer *)
(* [(1,1); (2,1); (2,2); (3,1); (3,2); (3,3)] *)

(* Simultaneous: for ... and ... *)
let s1 = [ (x, y) for x = 1 to 3 and y = 10 to 12 ]
(* [(1,10); (1,11); (1,12); (2,10); (2,11); (2,12); (3,10); (3,11); (3,12)] *)
(* 3 * 3 = 9 elements — NOT a zip; comprehensions cannot express zip *)
```

`[ (x, y) for x = 1 to 3 and y = 1 to x ]` is an error: `x` is not in
scope.

| Aspect | `for ... for ...` | `for ... and ...` |
|--------|-------------------|-------------------|
| Result size | Product | Product (same elements) |
| Re-evaluation | Inner re-evaluated each outer step | All evaluated once upfront |
| Dependencies | Inner can use outer | Independent (later iterators cannot reference earlier ones) |
| Optimization | Standard | Fixed-size array opt |

**Array optimization**: for an array comprehension with a single
`for ... and ...` clause the compiler pre-allocates the exact-size
array (`[| x + y for x = 1 to 1000 and y = 1 to 1000 |]` pre-allocates
1,000,000 elements). Not optimized: nested `for`
(`for x = 1 to 1000 for y = 1 to 1000` grows the array dynamically)
and any `when` clause (the size can't be predicted).

## Filter Clauses

```ocaml
let w1 = [ x for x = 1 to 100 when x mod 2 = 0 when x mod 3 = 0 ]
(* Multiple when: same as  when x mod 2 = 0 && x mod 3 = 0 *)

(* Position matters. Filter before inner loop: only even x, all y *)
let w2 = [ (x, y) for x = 1 to 10 when x mod 2 = 0 for y = 1 to 3 ]
(* Filter after inner loop: all x, only y > x *)
let w3 = [ (x, y) for x = 1 to 10 for y = 1 to 3 when y > x ]
```

## Evaluation Order

1. Clauses evaluated left to right
2. `for ... and ...` evaluates all sources before iterating
3. `when` checked before proceeding to right clauses
4. Body evaluated for each combination passing all filters

---

## Common Patterns

```ocaml
let map f lst = [ f x for x in lst ]
let map_array f arr = [| f x for x in arr |]
let filter pred lst = [ x for x in lst when pred x ]
let filter_map f lst =
  [ y for x in lst for y in (match f x with Some v -> [v] | None -> []) ]
let somes options =
  [ x for opt in options when Option.is_some opt
      for x in [Option.get opt] ]
let product xs ys = [ (x, y) for x in xs for y in ys ]
let flatten matrix = [ x for row in matrix for x in row ]
(* flatten [[1;2;3]; [4;5;6]; [7;8;9]] = [1; 2; 3; 4; 5; 6; 7; 8; 9] *)

(* Indexed iteration *)
let indexed items = [| (i, items.(i)) for i = 0 to Array.length items - 1 |]
(* indexed [|"a"; "b"; "c"; "d"|] = [|(0,"a"); (1,"b"); (2,"c"); (3,"d")|] *)
```

Not expressible as comprehensions:

- **Zip**: `and` produces the Cartesian product, never lockstep
  pairing. To pair two lists element-wise use `List.combine` /
  `List.map2` (or index into arrays as in `indexed`).
- **Enumerate**: use `List.mapi (fun i x -> (i, x)) lst`, or index an
  array as in `indexed`.
- **Take-while / early exit**: comprehensions don't short-circuit (no
  `break` or `return`), so every element is processed — write a
  recursive function instead.

---

## Performance Tips

1. **Prefer `and` over nested `for`** when iterators are independent — sources
   are evaluated once and array comprehensions can pre-allocate the exact size
2. **Put restrictive `when` clauses early** to skip work
3. **Array comprehensions** are generally faster than list (no cons overhead)
4. **For very large results**, consider alternative approaches (iterators, sequences)

## Limitations

Beyond the rules above (extension flag, matching global sources, no
unboxed types, partial patterns, no early exit):

- The result is always global (heap-allocated) — list comprehensions
  build intermediates on the local stack internally, but cannot
  produce a local list
- Nested comprehensions can be memory-intensive

See also: [SKILL.md](SKILL.md) for quick reference,
[SKILL-STACK-ALLOCATION.md](SKILL-STACK-ALLOCATION.md) for local list building.
