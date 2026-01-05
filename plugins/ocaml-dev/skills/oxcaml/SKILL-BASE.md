# OxCaml Base Library Extensions

Jane Street's Base library includes OxCaml-specific extensions for modes, local
allocation, and unboxed types. This guide covers Base features specific to
OxCaml that differ from standard OCaml Base.

## Mode-Aware Functions

Many Base functions have local-returning variants via `ppx_template`:

### List Operations

```ocaml
open Base

(* Standard - returns global *)
val List.map : 'a list -> f:('a -> 'b) -> 'b list

(* Local variant - returns local list *)
val List.map__local : 'a list -> f:('a -> 'b @ local) -> 'b list @ local

(* Usage *)
let process lst =
  let local_ mapped = List.map__local lst ~f:(fun x -> x + 1) in
  List.fold mapped ~init:0 ~f:(+)
```

### Common Local Variants

Most collection functions have `__local` variants:

```ocaml
List.map__local
List.filter__local
List.filter_map__local
List.concat_map__local
List.fold__local      (* f can return local *)
Array.map__local
Array.filter__local
```

### Instantiating via Templates

When using ppx_template in your code:

```ocaml
let%template[@mode m = (global, local)] my_map lst ~f =
  (List.map [@mode m]) lst ~f
```

---

## Option and Result with Modes

### Local Option Operations

```ocaml
(* Map over option, return local *)
val Option.map__local : 'a option -> f:('a -> 'b @ local) -> 'b option @ local

(* Bind with local *)
val Option.bind__local : 'a option -> f:('a -> 'b option @ local) -> 'b option @ local
```

### Result Operations

```ocaml
val Result.map__local : ('a, 'e) Result.t -> f:('a -> 'b @ local) -> ('b, 'e) Result.t @ local
val Result.map_error__local : ('a, 'e) Result.t -> f:('e -> 'f @ local) -> ('a, 'f) Result.t @ local
```

---

## Local Allocation Helpers

### Exn Module

```ocaml
(* Protect with local cleanup *)
val Exn.protect__local
  :  f:(unit -> 'a @ local)
  -> finally:(unit -> unit)
  -> 'a @ local

(* Usage *)
let with_resource f =
  let r = acquire () in
  Exn.protect__local
    ~f:(fun () -> f r)
    ~finally:(fun () -> release r)
```

### With_return

```ocaml
(* Early return with local value *)
val With_return.with_return__local
  : ('a With_return.return -> 'b @ local) -> 'b @ local
```

---

## Portable Functors

Many Base functors have portable variants for multicore:

### Comparable

```ocaml
(* Standard *)
module Comparable.Make (T : sig
  type t [@@deriving compare, sexp_of]
end) : Comparable.S with type t := T.t

(* Portable variant *)
module Comparable.Make__portable (T : sig
  type t [@@deriving compare, sexp_of] @@ portable
end) : Comparable.S with type t := T.t @@ portable
```

### Hashable

```ocaml
module Hashable.Make__portable (T : sig
  type t [@@deriving hash, compare, sexp_of] @@ portable
end) : Hashable.S with type t := T.t @@ portable
```

### Using %template.portable

```ocaml
module%template.portable My_type = struct
  type t = { x : int; y : int }
  [@@deriving compare, sexp_of, hash]

  include functor Comparable.Make
  include functor Hashable.Make
end
```

---

## Unboxed Numeric Operations

### Float_u in Base

```ocaml
open Base

(* Unboxed float operations available *)
module Float_u : sig
  type t = float#

  val add : t -> t -> t
  val sub : t -> t -> t
  val mul : t -> t -> t
  val div : t -> t -> t
  val neg : t -> t
  val abs : t -> t
  (* etc. *)
end
```

### Int32_u, Int64_u

```ocaml
module Int32_u : sig
  type t = int32#
  val add : t -> t -> t
  val sub : t -> t -> t
  (* etc. *)
end

module Int64_u : sig
  type t = int64#
  val add : t -> t -> t
  (* etc. *)
end
```

---

## Containers with Mode Support

### Array

```ocaml
(* Local iteration - closure can be local *)
val Array.iter__local : 'a array -> f:('a -> unit @ local) -> unit

(* Fold with local accumulator *)
val Array.fold__local
  :  'a array
  -> init:'acc
  -> f:('acc -> 'a -> 'acc @ local)
  -> 'acc
```

### Hashtbl

```ocaml
(* Find with local return *)
val Hashtbl.find__local : ('k, 'v) t -> 'k -> 'v option @ local

(* Iterate with local closure *)
val Hashtbl.iter__local : ('k, 'v) t -> f:(key:'k -> data:'v -> unit @ local) -> unit
```

### Map

```ocaml
val Map.find__local : ('k, 'v, 'cmp) t -> 'k -> 'v option @ local
val Map.fold__local
  :  ('k, 'v, _) t
  -> init:'acc
  -> f:(key:'k -> data:'v -> 'acc -> 'acc @ local)
  -> 'acc
```

---

## String and Bytes

### Local String Operations

```ocaml
(* Split returning local list *)
val String.split__local : string -> on:char -> string list @ local

(* Concat with local intermediate *)
val String.concat__local : ?sep:string -> string list -> string @ local
```

### Bytes with Modes

```ocaml
(* Create local bytes *)
val Bytes.create__local : int -> bytes @ local

(* Blit operations work with local *)
val Bytes.blit
  :  src:bytes @ local
  -> src_pos:int
  -> dst:bytes
  -> dst_pos:int
  -> len:int
  -> unit
```

---

## Sequence with Modes

```ocaml
(* Sequence that produces local elements *)
type 'a local_sequence

val Sequence.to_list__local : 'a Sequence.t -> 'a list @ local

val Sequence.fold__local
  :  'a Sequence.t
  -> init:'acc
  -> f:('acc -> 'a -> 'acc @ local)
  -> 'acc
```

---

## Applicative and Monad

### Local Applicative

```ocaml
module type Applicative_local = sig
  type 'a t

  val return : 'a -> 'a t
  val apply__local : ('a -> 'b @ local) t -> 'a t -> 'b t @ local
  val map__local : 'a t -> f:('a -> 'b @ local) -> 'b t @ local
end
```

### Local Monad

```ocaml
module type Monad_local = sig
  type 'a t

  val return : 'a -> 'a t
  val bind__local : 'a t -> f:('a -> 'b t @ local) -> 'b t @ local
  val map__local : 'a t -> f:('a -> 'b @ local) -> 'b t @ local
end
```

---

## Common Patterns

### Local List Processing Pipeline

```ocaml
let process_items items =
  let local_ filtered = List.filter__local items ~f:is_valid in
  let local_ mapped = List.map__local filtered ~f:transform in
  List.fold mapped ~init:0 ~f:(+)  (* Final result is global *)
```

### Portable Module Definition

```ocaml
module%template.portable My_key = struct
  module T = struct
    type t = string [@@deriving compare, sexp_of, hash]
  end
  include T
  include functor Comparable.Make
  include functor Hashable.Make
end
```

### With Local Intermediate Structures

```ocaml
let group_by items ~key =
  let local_ groups = Hashtbl.create__local (module String) in
  List.iter items ~f:(fun item ->
    Hashtbl.add_multi__local groups ~key:(key item) ~data:item
  );
  Hashtbl.to_alist groups  (* Convert to global at the end *)
```

---

## Migration Notes

When converting code to use OxCaml Base features:

1. **Look for `__local` variants** of functions you use
2. **Use `ppx_template`** for mode-polymorphic code
3. **Use `%template.portable`** for multicore-safe modules
4. **Check for unboxed numeric modules** (`Float_u`, `Int32_u`, etc.)
5. **Use `include functor`** to reduce boilerplate

See also: [SKILL-MODES.md](SKILL-MODES.md) for mode details,
[SKILL-CORE.md](SKILL-CORE.md) for Core-specific extensions.
