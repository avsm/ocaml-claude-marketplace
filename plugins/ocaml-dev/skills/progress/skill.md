---
name: progress
description: "Terminal progress bars and spinners using the OCaml progress library. Use when Claude needs to: (1) Add progress bars to long-running CLI operations, (2) Show download/upload progress with bytes and speed, (3) Create multi-line progress displays, (4) Integrate progress reporting with Eio concurrency"
---

# OCaml Progress Library

The `progress` library provides beautiful terminal progress bars, spinners, and multi-line displays for CLI applications.

## Installation

Add to `dune-project`:

```lisp
(depends
 (progress (>= 0.4)))
```

Add to library/executable `dune`:

```lisp
(libraries progress)
```

## Core Concepts

| Type | Purpose |
|------|---------|
| `Progress.Line.t` | A single progress line (bar, spinner, text) |
| `Progress.Multi.t` | Multiple lines combined |
| `Progress.Display.t` | Active display that can be updated |
| `Progress.Reporter.t` | Handle to report progress to a line |

## Basic Progress Bar

```ocaml
open Progress

let process_items items =
  let total = List.length items in
  let bar = Line.(list [
    spinner ();
    bar ~style:`UTF8 ~width:(`Fixed 40) total;
    count_to total;
  ]) in
  with_reporter bar (fun report ->
    List.iteri (fun i item ->
      process_item item;
      report (i + 1)
    ) items)
```

Output:
```
⠋ [████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 12/50
```

## Progress Bar with Bytes and Speed

For downloads/uploads, use `Int63` for large file sizes:

```ocaml
open Progress
module Int63 = Optint.Int63

let download_line ~total message =
  let open Line.Using_int63 in
  list [
    rpad 20 (const message);
    bytes;
    const " ";
    bytes_per_sec;
    const " ";
    bar ~style:`UTF8 ~width:(`Fixed 30) total;
    const " ";
    percentage_of total;
  ]

let download ~url ~size =
  let bar = download_line ~total:(Int63.of_int size) "Downloading..." in
  with_reporter bar (fun report ->
    fetch_with_progress url (fun bytes_read ->
      report (Int63.of_int bytes_read)))
```

Output:
```
Downloading...       12.5 MiB  2.3 MiB/s [██████████░░░░░░░░░░░░░░░░░░░░]  42%
```

## Multi-Line Progress Display

For operations with multiple concurrent tasks:

```ocaml
open Progress

let multi_download files =
  let lines = List.mapi (fun i file ->
    download_line ~total:(file_size file) (file_name file)
  ) files in
  let display = Multi.(list lines) in
  with_reporters display (fun reporters ->
    (* Each reporter corresponds to one line *)
    List.iter2 (fun reporter file ->
      download_file file (fun n -> reporter n)
    ) reporters files)
```

## Dynamic Line Addition

Add lines dynamically during execution:

```ocaml
open Progress

type t = {
  display : ((unit -> unit) -> unit, unit) Display.t;
  mutable line_count : int;
}

let init () =
  let header = Line.(constf "🐫 Processing files...") in
  let display = Display.start Multi.(line header) in
  { display; line_count = 0 }

let add_line t bar =
  let reporter = Display.add_line t.display bar in
  t.line_count <- t.line_count + 1;
  reporter

let finalise t =
  Display.finalise t.display
```

## Custom Colors

Use hex colors for branded progress bars:

```ocaml
open Progress

(* Gradient color palette *)
let colors = [|
  Color.hex "#1996f3";  (* Blue *)
  Color.hex "#06aeed";
  Color.hex "#10c6e6";
  Color.hex "#27dade";
  Color.hex "#3dead5";
  Color.hex "#52f5cb";  (* Cyan *)
  Color.hex "#66fcc2";
  Color.hex "#7dffb6";
  Color.hex "#92fda9";  (* Green *)
|]

let color_for_index i = colors.(i mod Array.length colors)

let colored_bar ~index ~total message =
  let color = color_for_index index in
  Line.(list [
    rpad 20 (const message);
    bar ~color ~style:`UTF8 total;
    percentage_of total;
  ])
```

## Spinners

For indeterminate operations:

```ocaml
open Progress

let with_spinner message f =
  let line = Line.(list [spinner (); const (" " ^ message)]) in
  with_reporter line (fun _report -> f ())

(* Usage *)
let () =
  with_spinner "Connecting to server..." (fun () ->
    connect_to_server ())
```

## Integration with Eio

For async progress updates, use a stream to communicate between fibers:

```ocaml
open Progress

type t = {
  stream : (unit -> unit) option Eio.Stream.t;
  display : ((unit -> unit) -> unit, unit) Display.t;
}

type reporter = {
  stream : (unit -> unit) option Eio.Stream.t;
  reporter : int Reporter.t option;
}

(* Report progress from any fiber *)
let report r value =
  match r.reporter with
  | None -> ()
  | Some reporter ->
      Eio.Stream.add r.stream
        (Some (fun () -> Reporter.report reporter value))

(* Process stream updates in display fiber *)
let rec process_stream ~sw stream =
  Eio.Switch.check sw;
  match Eio.Stream.take stream with
  | Some f -> f (); process_stream ~sw stream
  | None -> ()

let init ~sw =
  let stream = Eio.Stream.create max_int in
  let header = Line.(const "Processing...") in
  let display = Display.start Multi.(line header) in
  Eio.Fiber.fork ~sw (fun () -> process_stream ~sw stream);
  { stream; display }

let finalise t =
  Eio.Stream.add t.stream None;
  Display.finalise t.display
```

## Line Components Reference

| Component | Description |
|-----------|-------------|
| `bar ~style total` | Progress bar (styles: `UTF8, `ASCII) |
| `spinner ()` | Animated spinner |
| `count_to total` | "n/total" counter |
| `percentage_of total` | "42%" percentage |
| `const s` | Static string |
| `constf fmt` | Formatted static string |
| `elapsed ()` | Elapsed time |
| `eta total` | Estimated time remaining |
| `bytes` | Byte count (auto-scales to KiB, MiB, etc.) |
| `bytes_per_sec` | Transfer speed |
| `rpad n line` | Right-pad to n characters |
| `lpad n line` | Left-pad to n characters |
| `spacer n` | Fixed-width space |
| `list lines` | Combine multiple components |
| `(++)` | Infix combine: `a ++ b` |

## Bar Styles

```ocaml
(* UTF-8 style (default, looks best) *)
Line.bar ~style:`UTF8 total
(* Output: [████████░░░░░░░░] *)

(* ASCII style (for limited terminals) *)
Line.bar ~style:`ASCII total
(* Output: [########........] *)

(* Custom width *)
Line.bar ~style:`UTF8 ~width:(`Fixed 50) total
Line.bar ~style:`UTF8 ~width:(`Expand) total  (* fill available space *)
```

## Complete CLI Example

```ocaml
open Cmdliner
open Progress

let download_files files =
  let total_files = List.length files in
  let header = Line.(constf "📦 Downloading %d files" total_files) in
  let display = Display.start Multi.(line header) in

  List.iteri (fun i file ->
    let size = Int63.of_int (file_size file) in
    let color = color_for_index i in
    let bar = Line.Using_int63.(list [
      rpad 25 (const (file_name file));
      bytes;
      bar ~color ~style:`UTF8 ~width:(`Fixed 30) size;
      percentage_of size;
    ]) in
    let reporter = Display.add_line display bar in
    download_file file (fun bytes ->
      Reporter.report reporter (Int63.of_int bytes));
    Reporter.finalise reporter
  ) files;

  Display.finalise display;
  Fmt.pr "✓ Downloaded %d files@." total_files

let cmd =
  let info = Cmd.info "download" ~doc:"Download files with progress" in
  Cmd.v info Term.(const download_files $ files_arg)
```

## Tips

1. **Use `Int63` for bytes** - Regular `int` overflows at 2GB on 32-bit systems
2. **Fork display updates** - With Eio, run display in separate fiber
3. **Finalise reporters** - Always call `Reporter.finalise` when done
4. **Check terminal width** - Use `(`Expand)` for responsive bars
5. **Provide fallback** - Disable progress when stdout isn't a TTY:

```ocaml
let with_progress ~enabled bar f =
  if enabled && Unix.isatty Unix.stdout then
    with_reporter bar f
  else
    f (fun _ -> ())
```
