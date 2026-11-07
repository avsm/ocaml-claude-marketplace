---
name: oxcaml-core
description: "OxCaml Core library extensions including Bigstring, Time_ns, Core_unix, Bin_prot/Sexp local variants, capsules and portable data with modal support"
---

# OxCaml Core Library Extensions

For `core` / `core_unix` `v0.18~preview.133.03+642` (OxCaml
5.4.0-ox9). Signatures are shown module-qualified (`val
Bigstring.length : ...`) as documentation shorthand — they are not
compilable `val` items. If a call fails to typecheck, check the `.mli`
under `$(opam var lib)/core/`.

**How Core exposes modes.** Unlike Base's collections, most Core I/O
functions do *not* come in `__local` variants. Instead the ordinary
function is annotated to **accept** local (and often `read`/`shared`)
arguments, e.g. `Bigstring.get_int32_le : t @ local read -> pos:int ->
int`. Stack-allocated *results* use the `__stack` suffix
(`(f [@alloc stack])`), as in Base — see
[SKILL-BASE.md](SKILL-BASE.md) for the naming convention.

## Channels, Command, Core_unix

`In_channel`/`Out_channel` have no mode variants; buffers passed to
them must be global (reuse one heap buffer across iterations). For a
stack-allocated buffer, read through `Core_unix.read` instead.

`Filename_unix` and `Command_unix.run` need the dune libraries
`core_unix.filename_unix` and `core_unix.command_unix`. Inside a
`Command` body, stack-allocated intermediates work as usual:
`let local_ processed = (List.map [@alloc stack]) files ~f:process_file
in output_results processed [@nontail]` (`output_results` must accept
its argument `@ local`).

The ordinary `Core_unix` syscall wrappers accept **local buffers**
directly — there are no `read__local`/`recv__local` variants:

```ocaml
val Core_unix.read
  : ?restart:bool -> ?pos:int -> ?len:int -> File_descr.t -> buf:Bytes.t @ local -> int
val Core_unix.write
  : ?pos:int -> ?len:int -> File_descr.t -> buf:Bytes.t @ local read -> int
val Core_unix.recv
  : File_descr.t -> buf:Bytes.t @ local -> pos:int -> len:int -> mode:msg_flag list -> int
val Core_unix.recvfrom
  : File_descr.t -> buf:Bytes.t @ local -> pos:int -> len:int -> mode:msg_flag list
  -> int * sockaddr

(* stat/lstat/fstat return an ordinary heap-allocated stats record *)
val Core_unix.stat : string -> stats
```

So a read loop can use a stack-allocated buffer:

```ocaml
open Core

let read_fd fd =
  let local_ buf = Bytes.create_local 8192 in
  let n = Core_unix.read fd ~buf in
  process_chunk buf ~len:n [@nontail]
```

## Bigstring

Nearly every accessor takes the bigstring `@ local read` (setters
`@ local`), so they work on local, shared and read-only views.

```ocaml
module Bigstring : sig
  (* Bigstring is a char bigarray with C layout *)
  type t = (char, int8_unsigned_elt, c_layout) Bigarray.Array1.t

  (* Creation *)
  val create : int -> t @ unique
  val init : int -> f:(int -> char) -> t

  (* From existing data *)
  val of_string : ?pos:int -> ?len:int -> string @ local -> t
  val of_bytes : ?pos:int -> ?len:int -> bytes @ local read -> t

  (* Sub-view (shares memory) *)
  val sub_shared : ?pos:int -> ?len:int -> t -> t
  val sub_shared_local : ?pos:int -> ?len:int -> t @ local -> t @ local
end
```

The data always lives in C-heap memory; only the OCaml wrapper can be
local. Compared with `Bytes`: C heap rather than OCaml heap (avoids GC
pressure), size limited by the system rather than ~16MB (32-bit), can
wrap mmap'd memory; both have fast optimized access primitives and
`int`-returning multi-byte getters (no boxing); only `Bytes` can be
created local (`Bytes.create_local`).

### Access

```ocaml
(* Single byte access *)
external get : (t[@local_opt]) @ read -> int -> char
external set : (t[@local_opt]) -> int -> char -> unit
(* also unsafe_get, unsafe_set *)

(* Multi-byte access returns a plain int - no boxing *)
val get_int8 : t @ local read -> pos:int -> int        (* also get_uint8 *)
val get_int16_le : t @ local read -> pos:int -> int
val get_int32_le : t @ local read -> pos:int -> int
val get_int64_le_exn : t @ local read -> pos:int -> int    (* raises if > 63 bits *)
val get_int64_le_trunc : t @ local read -> pos:int -> int  (* truncates *)
(* big-endian: get_int16_be, get_int32_be, get_int64_be_exn, … *)

(* Full-width boxed results *)
val get_int32_t_le : t @ local read -> pos:int -> Int32.t
val get_int64_t_le : t @ local read -> pos:int -> Int64.t

(* Setters follow the same pattern *)
val set_int32_le_exn : t @ local -> pos:int -> int -> unit
val set_int64_le : t @ local -> pos:int -> int -> unit
val set_int64_t_le : t @ local -> pos:int -> Int64.t @ local -> unit
```

There are **no `*_unboxed` accessors**. For zero-allocation parsing,
use the `int`-returning getters above (an `int` is unboxed already;
they satisfy `[@zero_alloc]`):

```ocaml
open Core

(* Parse binary protocol without allocation *)
type header = #{ magic : int; version : int; length : int; timestamp : int }

let[@zero_alloc] parse_header (bigstring @ local) ~pos : header =
  let magic = Bigstring.get_int32_le bigstring ~pos in
  let version = Bigstring.get_int16_le bigstring ~pos:(pos + 4) in
  let length = Bigstring.get_int32_le bigstring ~pos:(pos + 6) in
  let timestamp = Bigstring.get_int64_le_trunc bigstring ~pos:(pos + 10) in
  #{ magic; version; length; timestamp }  (* unboxed record *)
```

If you need a full-width unboxed value, read into a local box and
unbox it (`Int64_u`/`Int32_u` come from the `unboxed` package):

```ocaml
open Core

let read_i64 (b : Bigstring.t @ local) ~pos : int64_u =
  Int64_u.of_int64 (Bigstring.Local.get_int64_t_le b ~pos)
(* Bigstring.Local.get_int64_t_le : t @ local read -> pos:int -> Int64.t @ local *)
```

### Bulk Operations, Conversion, Searching

```ocaml
(* Memory operations *)
val length : t @ local immutable -> int
val memset : t @ local -> pos:int -> len:int -> char -> unit
val memcmp : t @ local read -> pos1:int -> t @ local read -> pos2:int -> len:int -> int
val blit
  : src:t @ local read -> src_pos:int -> dst:t @ local -> dst_pos:int -> len:int -> unit

(* With bytes/string: blit submodules *)
Bigstring.From_string.blit   (* string -> bigstring *)
Bigstring.To_string.blit     (* bigstring -> bytes, plus sub/subo -> string *)
Bigstring.From_bytes.blit
Bigstring.To_bytes.blit
```

```ocaml
(* To string/bytes - copies data *)
val to_string : ?pos:int -> ?len:int -> t @ local read -> string
val to_bytes : ?pos:int -> ?len:int -> t @ local read -> bytes

(* Stack-allocated result *)
val to_string__stack : ?pos:int -> ?len:int -> t @ local read -> string @ local

(* Substring extraction *)
val get_string : t @ local read -> pos:int -> len:int -> string
val Local.get_string : t @ local read -> pos:int -> len:int -> string @ local
```

```ocaml
val find : ?pos:int -> ?len:int -> char -> t @ local read -> int option
```

There are no `iter`/`fold` closures over bigstrings; write an index
loop with `Bigstring.get`.

```ocaml
open Core

(* Local string extraction for temporary use *)
let find_field bigstring ~field_pos ~field_len =
  let local_ field_str =
    Bigstring.Local.get_string bigstring ~pos:field_pos ~len:field_len
  in
  lookup_field field_str [@nontail]  (* result escapes, string doesn't *)
```

### I/O (`Bigstring_unix`)

Channel and file-descriptor I/O on bigstrings lives in
**`Bigstring_unix`** (dune library `core_unix.bigstring_unix`), not in
`In_channel`/`Out_channel`/`Core.Bigstring`. All take the bigstring
`@ local` (writes: `@ local shared`):

```ocaml
(* Read directly into a bigstring - no copy *)
val Bigstring_unix.really_input
  : In_channel.t -> ?pos:int -> ?len:int -> Bigstring.t @ local -> unit
val Bigstring_unix.read
  : ?min_len:int -> Core_unix.File_descr.t -> ?pos:int -> ?len:int
  -> Bigstring.t @ local -> int

(* Write from a bigstring *)
val Bigstring_unix.really_output
  : Out_channel.t -> ?pos:int -> ?len:int -> Bigstring.t @ local shared -> unit
val Bigstring_unix.write
  : ?min_len:int -> Core_unix.File_descr.t -> ?pos:int -> ?len:int
  -> Bigstring.t @ local shared -> int

(* also input, output, really_read, really_write, pread, pwrite *)
```

## Time Without Allocation

`Time_ns.t` and `Time_ns.Span.t` are `immediate64` (an `Int63`: unboxed
on 64-bit targets), so the core operations are already allocation-free
and annotated as such — there is no separate unboxed time type.

```ocaml
val Time_ns.now : unit -> t                              (* [@@zero_alloc] *)
val Time_ns.diff : t -> t -> Span.t                      (* [@@zero_alloc strict] *)
val Time_ns.add : t -> Span.t -> t                       (* [@@zero_alloc strict] *)
val Time_ns.sub : t -> Span.t -> t                       (* [@@zero_alloc strict] *)
val Time_ns.to_int63_ns_since_epoch : t -> Int63.t       (* [@@zero_alloc strict] *)
val Time_ns.of_int63_ns_since_epoch : Int63.t -> t       (* [@@zero_alloc strict] *)
val Time_ns.min__local : t @ local -> t @ local -> t @ local   (* also max__local *)
```

```ocaml
val Time_ns.Span.to_int63_ns : t -> Int63.t
val Time_ns.Span.of_int63_ns : Int63.t -> t
val Time_ns.Span.to_int_ns : t -> int                    (* [@@zero_alloc] *)
val Time_ns.Span.of_int_ns : int -> t

(* Scale by an unboxed float without boxing *)
val Time_ns.Span.scale_u : t -> float# -> t              (* [@@zero_alloc] *)
```

## Bin_prot, Sexp, Or_error

Binary protocol serialization of **local values**: the primitive
writers/readers have `__local` instances, and the deriver generates
them on request:

```ocaml
open Core

type t = { a : int; b : string } [@@deriving bin_io ~localize]

(* Generated, in addition to the usual bin_* functions:
   bin_size_t__local, bin_write_t__local,
   bin_writer_t__local : t Bin_prot.Type_class.writer__local *)

(* Primitive instances *)
let w : int Bin_prot.Write.writer__local = Bin_prot.Write.bin_write_int__local
```

`Bigstring.write_bin_prot_known_size__local` writes a local value with
such a writer.

Stack-allocated sexp output (parsing with `Sexp.of_string` always
produces a heap value):

```ocaml
val Sexp.to_string__stack : t @ local -> string @ local
val Sexp.to_string_hum__stack : ?indent:int -> ?max_width:int -> t @ local -> string @ local
val Sexp.to_string_mach__stack : t @ local -> string @ local
(* equivalently (Sexp.to_string [@alloc stack]) *)
```

Deriver flags `sexp_of ~stackify` (`sexp_of_t__stack : t @ local ->
Sexp.t @ local`), `compare ~localize`, `equal ~localize`, `globalize`:
see "Deriving with Modes" in [SKILL-BASE.md](SKILL-BASE.md).

`Or_error.map__local` / `bind__local` and the domain-crossing
`Or_error.Portable` are described in [SKILL-BASE.md](SKILL-BASE.md).
Errors created with `Or_error.error_s` / `error_string` / `errorf` are
ordinary heap values: an `Error.t` is a heap value (often lazy), so
there are no local-message variants.

## Async

`Async_kernel`'s `Deferred` and `Pipe` have **no mode or
portability variants** — no `__local`, no `__portable`. Async code is
single-domain; closures passed to `Deferred.map`, `Pipe.fold`, etc.
are ordinary global, nonportable functions. Use the capsule/`Portable`
APIs below (or the `parallel` / `await` packages) for multicore work,
and `await_in_async` (or `Async_kernel.Ivar.Portable`) to bridge.

## Portable Data and Capsules

`Core.Capsule` is `Portable.Capsule` (packages `portable`, `capsule`).
Capsules let mutable state be shared across domains safely.

### Frozen: Build Mutably, Share Read-Only

```ocaml
open Core

let table : int array Capsule.Frozen.t =
  Capsule.Frozen.create (fun () ->
    let t = Array.create ~len:4 0 in
    t.(0) <- 42;
    t)

let lookup i =
  let (t @ shared) = Capsule.Frozen.unwrap table in
  (Array.get [@mode shared]) t i
```

The unwrapped value is `shared`, so use the shared-mode instances of
readers: plain `t.(i)` fails with "This value is shared but is
expected to be uncontended". For a hash table, create it with
`(Hashtbl.create [@mode portable]) (module String)` and read it with
`(Hashtbl.find [@mode shared])`.

### Access Tokens and Scoped Capsules

```ocaml
open Core

let f () =
  let (P access) = Capsule.Access.current () in
  let d = Capsule.Data.wrap ~access (ref 0) in
  let r = Capsule.Data.unwrap ~access d in
  incr r;
  !r

let s x =
  Capsule.Scoped.with_ (ref x) ~f:(fun s ->
    Capsule.Scoped.iter s ~f:(fun r -> incr r);
    Capsule.Scoped.get s ~f:(fun r -> !r))
```

Surface: `Capsule.Access` (`current`, `box`, `unbox`), `Data`
(`create`, `wrap`, `unwrap`, `both`, `fst`, `snd`, …), `Frozen`,
`Owned` (`create`, `freeze`, `with_`, …), `Scoped`, `Initial`,
`Portable_via_contended`. The key/password layer is `Capsule.Prim`.

- The old `*_unique` / `*_once` / `*_local` function variants in
  `Capsule.Prim` (`Data.wrap_unique`, `Data.create_once`,
  `access_local`, `Key.box_aliased`, …) no longer exist; use the
  **mode-templated instances**: `wrap_unique` is
  `(Capsule.Prim.Data.wrap [@mode unique])`, `wrap_once_unique` is
  `(… wrap [@mode once unique])`, and local variants are e.g.
  `(Capsule.Data.unwrap [@mode local])`.
- `Capsule.Prim.Key.box` / `unbox` have **no unsuffixed instance**
  (`Unbound value Capsule.Prim.Key.box`): every instance names both
  modes, so ask with `.explicit` —
  `(Capsule.Prim.Key.box [@mode.explicit global unique])` (the old
  `Key.box`) or `[@mode.explicit global aliased]` (the old
  `Key.box_aliased`); `local` variants also exist.
- `Capsule.Blocking_sync` (mutexes/condition variables) is
  alert-deprecated in favour of the synchronisation primitives in the
  `await` package; its `Mutex.with_lock` callbacks also need to be
  `unyielding`.
- `Capsule.create` does not exist; use `Capsule.Prim.create`,
  `Capsule.Frozen.create`, `Capsule.Owned.create` or
  `Capsule.Scoped.with_`.

## Spawning Domains, `parallel`, `await`

`Domain.spawn` carries alerts; the supported low-level entry point is
`Multicore` (module of the compiler's `threads` library), with
`Portable.Atomic` for shared counters:

```ocaml
(* dune: (libraries base portable threads) (preprocess (pps ppx_jane)) *)
open Base
open Portable

let () =
  let counter = Atomic.make 0 in
  let finished = Atomic.make 0 in
  let workers = 4 in
  for _ = 1 to workers do
    match
      Multicore.spawn
        (fun () ->
          for _ = 1 to 1000 do Atomic.incr counter done;
          Atomic.incr finished)
        ()
    with
    | Spawned -> ()
    | Failed ((), exn, bt) -> Stdlib.Printexc.raise_with_backtrace exn bt
  done;
  while Atomic.get finished < workers do Stdlib.Domain.cpu_relax () done;
  Stdlib.Printf.printf "counter=%d\n" (Atomic.get counter)
```

For structured parallelism use the `parallel` package
(`Parallel.fork_join2`, run under `Parallel_scheduler.with_parallel`);
for locks use `await`. Both are separate opam packages at
`v0.18~preview.133.03+642` (requiring `oxcaml-compiler >=
5.4.0-ox9`):

```ocaml
(* dune: (libraries parallel parallel.scheduler) *)
let rec fib parallel n =
  match n with
  | 0 | 1 -> 1
  | n ->
    let #(a, b) =
      Parallel.fork_join2 parallel
        (fun parallel -> fib parallel (n - 1))
        (fun parallel -> fib parallel (n - 2))
    in
    a + b

let fib_parallel n =
  Parallel_scheduler.with_parallel (fun parallel -> fib parallel n)
```

Mutex-protected shared state lives in `Await.Capsule` (there is no
`Await.Mutex`): `Await.Capsule.Sync.With_mutex` pairs a value in its
own capsule with a mutex, and `Await.Capsule.Sync.Mutex` is the bare
lock (`create ()`, `with_lock`). Every operation takes a `Sync.t`
capability — `Parallel.sync par` inside the scheduler,
`Await.Sync.blocking` outside one:

```ocaml
(* dune: (libraries await parallel parallel.scheduler) *)
module Guarded = Await.Capsule.Sync.With_mutex

(* A counter in its own capsule, protected by a mutex *)
let total = Guarded.create (fun () -> ref 0)

let add par n =
  (* [@nontail]: the Sync.t is local, so this must not be a tail call *)
  Guarded.with_lock (Parallel.sync par) total ~f:(fun _sync r -> r := !r + n)
  [@nontail]

let () =
  Parallel_scheduler.with_parallel (fun par ->
    let #((), ()) =
      Parallel.fork_join2 par (fun par -> add par 1) (fun par -> add par 2)
    in
    ());
  (* outside a scheduler, block the OS thread *)
  let v = Guarded.with_lock Await.Sync.blocking total ~f:(fun _sync r -> !r) in
  Printf.printf "total=%d\n" v
```

The lower-level key-based mutexes are `Await.Sync.Mutex` and
`Await.Await.Mutex` (`create key`, `with_access`, `with_password`).

## Gotchas

- **Unboxed number types** are `int32_u` / `int64_u` / `nativeint_u` /
  `float32_u` (the `int32#`… spellings no longer exist; `float#` is
  unchanged). `[@@deriving sexp_of]` on a field written as bare
  `int64_u` fails (`Unbound value sexp_of_int64_u`) — write the field
  type as `Int64_u.t` (package `unboxed`).
- `Int32_u.to_int` / `Int64_u.to_int` return `int option`; use
  `to_int_exn` or `to_int_trunc`.
- **Effect-handler syntax under ppx**: `match … with effect E, k -> …`
  fails in files preprocessed by `ppx_jane` (`migration error: effect
  patterns is not supported before OCaml 5.02`). Keep handlers in a
  ppx-free file or use `Effect.Deep.try_with`.
- Portable functor instances (`Comparable.Make [@mode portable]`) and
  the unboxed-number modules: see [SKILL-BASE.md](SKILL-BASE.md).

See also: [SKILL-BASE.md](SKILL-BASE.md) for Base extensions,
[SKILL-ZERO-ALLOC.md](SKILL-ZERO-ALLOC.md) for allocation-free patterns.
