# OxCaml Core Library Extensions

Jane Street's Core library builds on Base with additional OxCaml-specific
features for I/O, concurrency, and system programming.

## Mode-Aware I/O

### In_channel and Out_channel

```ocaml
open Core

(* Read with local buffer *)
val In_channel.input__local : t -> buf:bytes @ local -> pos:int -> len:int -> int

(* Fold over lines with local accumulator *)
val In_channel.fold_lines__local
  :  t
  -> init:'acc
  -> f:('acc -> string -> 'acc @ local)
  -> 'acc
```

### Bigstring I/O

```ocaml
(* Read directly into bigstring - no copy *)
val In_channel.really_input_bigstring
  :  t
  -> Bigstring.t
  -> pos:int
  -> len:int
  -> unit

(* Write from bigstring *)
val Out_channel.output_bigstring
  :  t
  -> Bigstring.t
  -> pos:int
  -> len:int
  -> unit
```

---

## Time with Unboxed Types

### Time_ns

```ocaml
module Time_ns : sig
  (* Unboxed time representation *)
  type t  (* boxed *)
  type t_unboxed = int64#  (* unboxed nanoseconds since epoch *)

  val to_int63_ns_since_epoch : t -> Int63.t
  val of_int63_ns_since_epoch : Int63.t -> t

  (* Unboxed operations *)
  module Unboxed : sig
    val now : unit -> t_unboxed
    val diff : t_unboxed -> t_unboxed -> int64#
    val add : t_unboxed -> int64# -> t_unboxed
  end
end
```

### Time_ns.Span

```ocaml
module Time_ns.Span : sig
  type t

  (* Unboxed span operations *)
  val to_int63_ns : t -> Int63.t
  val of_int63_ns : Int63.t -> t

  (* Direct nanosecond access *)
  val to_ns_unboxed : t -> int64#
  val of_ns_unboxed : int64# -> t
end
```

---

## Command with Modes

### Local Argument Parsing

```ocaml
open Core

let command =
  Command.basic
    ~summary:"Process files"
    (let%map_open.Command
       files = anon (sequence ("FILE" %: Filename_unix.arg_type))
     and verbose = flag "-v" no_arg ~doc:"Verbose output"
     in
     fun () ->
       (* Argument processing can use local *)
       let local_ processed = List.map__local files ~f:process_file in
       output_results processed)
```

---

## Iobuf with Modes

Zero-copy buffer manipulation:

```ocaml
module Iobuf : sig
  type ('rw, 'seek) t

  (* Create local iobuf *)
  val create__local : len:int -> (read_write, seek) t @ local

  (* Consume with local return *)
  val Consume.stringo__local
    :  (read, _) t
    -> len:int
    -> string @ local

  (* Peek without consuming *)
  val Peek.int64_le : (read, _) t -> pos:int -> int64
  val Peek.int64_le_unboxed : (read, _) t -> pos:int -> int64#
end
```

### Iobuf Patterns

```ocaml
let parse_packet iobuf =
  (* Peek header without consuming *)
  let msg_type = Iobuf.Peek.int8 iobuf ~pos:0 in
  let msg_len = Iobuf.Peek.int32_le iobuf ~pos:1 in

  (* Consume the message *)
  let local_ payload = Iobuf.Consume.stringo__local iobuf ~len:msg_len in
  process_message msg_type payload
```

---

## Core_unix with Modes

### File Operations

```ocaml
module Core_unix : sig
  (* Read with local buffer *)
  val read__local
    :  File_descr.t
    -> buf:bytes @ local
    -> pos:int
    -> len:int
    -> int

  (* Stat returning local record *)
  val stat__local : string -> stats @ local
  val lstat__local : string -> stats @ local
end
```

### Socket Operations

```ocaml
(* Recv with local buffer *)
val recv__local
  :  File_descr.t
  -> buf:bytes @ local
  -> pos:int
  -> len:int
  -> recv_flag list
  -> int

(* Recvfrom returning local address *)
val recvfrom__local
  :  File_descr.t
  -> buf:bytes @ local
  -> pos:int
  -> len:int
  -> recv_flag list
  -> int * sockaddr @ local
```

---

## Async with Modes (Async_kernel)

### Deferred with Local

```ocaml
module Deferred : sig
  (* Map with local function *)
  val map__local : 'a t -> f:('a -> 'b @ local) -> 'b t @ local

  (* Bind with local continuation *)
  val bind__local : 'a t -> f:('a -> 'b t @ local) -> 'b t @ local
end
```

### Pipe with Modes

```ocaml
module Pipe : sig
  (* Read with local return *)
  val read__local : 'a Reader.t -> [ `Ok of 'a | `Eof ] @ local Deferred.t

  (* Fold with local accumulator *)
  val fold__local
    :  'a Reader.t
    -> init:'acc
    -> f:('acc -> 'a -> 'acc @ local Deferred.t)
    -> 'acc Deferred.t
end
```

---

## Portable Async

For multicore Async:

```ocaml
(* Portable job scheduling *)
module Scheduler : sig
  val schedule__portable
    :  (unit -> unit) @ portable
    -> unit
end

(* Portable deferred operations *)
val Deferred.map__portable
  :  'a t
  -> f:('a -> 'b) @ portable
  -> 'b t @ portable
```

---

## Bigstring Extensions

### Unboxed Access

```ocaml
module Bigstring : sig
  (* Unboxed getters - no allocation *)
  val get_int64_le_unboxed : t -> pos:int -> int64#
  val get_int32_le_unboxed : t -> pos:int -> int32#
  val get_float_unboxed : t -> pos:int -> float#

  (* Unboxed setters *)
  val set_int64_le_unboxed : t -> pos:int -> int64# -> unit
  val set_int32_le_unboxed : t -> pos:int -> int32# -> unit
  val set_float_unboxed : t -> pos:int -> float# -> unit
end
```

### Zero-Copy Patterns

```ocaml
let process_binary_data bigstring =
  (* Read header without boxing *)
  let magic = Bigstring.get_int32_le_unboxed bigstring ~pos:0 in
  let length = Bigstring.get_int32_le_unboxed bigstring ~pos:4 in
  let timestamp = Bigstring.get_int64_le_unboxed bigstring ~pos:8 in

  (* Process without allocation *)
  if Int32_u.equal magic expected_magic then
    process_payload bigstring ~pos:16 ~len:(Int32_u.to_int length)
  else
    Error `Invalid_magic
```

---

## Bin_prot with Modes

Binary protocol serialization:

```ocaml
module Bin_prot : sig
  (* Write to local buffer *)
  val write__local : 'a Type_class.writer -> buf:bytes @ local -> 'a -> int

  (* Read with local intermediate *)
  val read__local
    :  'a Type_class.reader
    -> buf:bytes @ local
    -> pos_ref:int ref
    -> 'a
end
```

---

## Sexp with Modes

### Local Sexp Operations

```ocaml
module Sexp : sig
  (* Parse to local sexp *)
  val of_string__local : string -> t @ local

  (* Convert with local intermediate *)
  val to_string__local : t -> string @ local
end
```

### Sexp_of with Local

```ocaml
(* For types with local sexp_of *)
type t [@@deriving sexp_of__local]

val sexp_of_t__local : t -> Sexp.t @ local
```

---

## Error Handling

### Or_error with Modes

```ocaml
module Or_error : sig
  val map__local : 'a t -> f:('a -> 'b @ local) -> 'b t @ local
  val bind__local : 'a t -> f:('a -> 'b t @ local) -> 'b t @ local

  (* Error creation *)
  val error_s__local : Sexp.t @ local -> _ t
end
```

### Error with Local Message

```ocaml
let validate x =
  if x < 0 then
    let local_ msg = sprintf "Invalid value: %d" x in
    Or_error.error_string__local msg
  else
    Ok x
```

---

## Common Patterns

### Zero-Copy Network Processing

```ocaml
let handle_connection fd =
  let buf = Bigstring.create 4096 in
  let rec loop () =
    match Core_unix.read_bigstring fd buf ~pos:0 ~len:4096 with
    | 0 -> ()
    | n ->
      (* Process without copying *)
      let msg_type = Bigstring.get_int8 buf ~pos:0 in
      let payload_len = Bigstring.get_int32_le_unboxed buf ~pos:1 in
      process_message buf ~msg_type ~len:(Int32_u.to_int payload_len);
      loop ()
  in
  loop ()
```

### Portable Async Pipeline

```ocaml
let%template.portable process_stream reader =
  Pipe.fold reader ~init:0 ~f:(fun count item ->
    let result = process_item item in
    log_result result;
    return (count + 1)
  )
```

### Local Buffer Reuse

```ocaml
let process_many_files files =
  (* Reuse local buffer across iterations *)
  let local_ buf = Bytes.create 8192 in
  List.iter files ~f:(fun filename ->
    In_channel.with_file filename ~f:(fun ic ->
      let n = In_channel.input__local ic ~buf ~pos:0 ~len:8192 in
      process_chunk buf ~len:n
    )
  )
```

---

## Migration Checklist

When updating Core code for OxCaml:

1. **Replace heap buffers with local** where possible
2. **Use unboxed Bigstring accessors** for binary data
3. **Use `__local` variants** for intermediate collections
4. **Add `@@portable`** annotations for multicore code
5. **Use `Iobuf`** for zero-copy buffer manipulation
6. **Use unboxed Time_ns** operations in hot paths

See also: [SKILL-BASE.md](SKILL-BASE.md) for Base extensions,
[SKILL-ZERO-ALLOC.md](SKILL-ZERO-ALLOC.md) for allocation-free patterns.
