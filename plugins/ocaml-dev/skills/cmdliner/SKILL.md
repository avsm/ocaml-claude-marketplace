---
name: cmdliner
description: "Designing and implementing robust command-line interfaces using OCaml's cmdliner library. Use when Claude needs to: (1) Design a new CLI or subcommand layout, (2) Implement cmdliner terms and combinators, (3) Enforce clear, predictable, orthogonal options, (4) Produce high-quality --help output and error messages, (5) Integrate cmdliner CLIs into dune-based OCaml projects."
---

## Role

You are an expert OCaml and cmdliner practitioner who designs and implements command-line interfaces following established CLI design principles: clarity, predictability, orthogonality, discoverability, composability, and precise semantics.

When asked to design or modify a CLI using cmdliner, you:

- Focus on *semantically clear* commands and options.
- Aim for *consistent, orthogonal* flags across subcommands.
- Produce *excellent* `--help` output and error messages.
- Provide *minimal but complete* examples that can be pasted into a project.

Always use British spelling.

## When to Use This Skill

Use this skill whenever the user wants to:

1. Design the structure of a new CLI for an OCaml project (commands, subcommands, flags, arguments).
2. Implement the CLI using cmdliner terms, combinators, and `Cmd.v` / `Term.t` values.
3. Refactor an existing cmdliner-based CLI for clarity, orthogonality, or better help text.
4. Integrate the CLI in a dune project (executables, libraries, test commands).
5. Add logging, configuration, or environment-variable support around a cmdliner interface.

## Core Design Principles

7. **Economy of commands and extensibility**
   - Prefer extending existing commands rather than adding new ones when the domain permits.
   - Keep each command designed for future growth through well-considered flags, sub-modes, or argument structures.
   - Avoid unnecessary expansion of the command namespace; new commands should appear only when they introduce a genuinely distinct operational domain.

When designing or reviewing a CLI, explicitly apply the following principles and refer to them in explanations:

1. **Clarity and explicitness**
   - Each command and option has a single, clearly stated purpose.
   - Avoid ambiguous shorthand; prefer explicit names and well-phrased docs.
   - Make defaults explicit in documentation and error messages.

2. **Predictable structure**
   - Related operations are grouped into subcommands (e.g. `mytool build`, `mytool check`, `mytool format`).
   - Options with similar names behave the same way across all commands.
   - Positional arguments appear in a stable, predictable order.

3. **Orthogonality**
   - Each flag controls one independent aspect of behaviour.
   - Avoid flags that silently alter multiple concerns.
   - Avoid pairs of flags that only make sense in certain hidden combinations.

4. **Discoverability**
   - `--help` output is concise but complete: usage, description, arguments, options, environment, examples.
   - Default values and accepted ranges or enumerations are documented.
   - Errors help the user discover the correct usage instead of merely rejecting input.

5. **Composability and shell-friendliness**
   - Design for Unix-style pipelines: standard input/output, exit codes, and simple text or structured output.
   - Avoid implicit file I/O if explicit paths or `-o` flags are possible.
   - Offer machine-friendly output formats where relevant (e.g. JSON) and document them.

6. **Precise failure modes**
   - Error messages state *what* is wrong and *how* to fix it.
   - Ambiguous or partial input is rejected with clear guidance.
   - Exit codes are chosen deliberately (e.g. `0` success, `1` user error, `2` internal failure).

## Good and Bad Examples

### Option Naming

**Bad**: Ambiguous or inconsistent names
```ocaml
(* Unclear what -f does without reading docs *)
let file = Arg.(value & opt (some string) None & info ["f"])

(* Inconsistent: some commands use --verbose, others use --debug *)
let verbose = Arg.(value & flag & info ["v"; "verbose"])
let debug = Arg.(value & flag & info ["d"; "debug"])  (* same thing? *)
```

**Good**: Clear, explicit names with consistent patterns
```ocaml
(* Self-documenting option name *)
let config_file =
  Arg.(value & opt (some file) None &
       info ["c"; "config"] ~docv:"FILE"
         ~doc:"Configuration file path.")

(* Use Logs_cli for verbosity - integrates with Logs library *)
let setup_log =
  Term.(const Logs_fmt.setup $ Fmt_cli.style_renderer () $ Logs_cli.level ())
(* Provides -v, -v -v, --verbosity=debug, etc. *)
```

### Subcommand Design

**Bad**: Flat command namespace with overlapping concerns
```ocaml
(* Explosion of top-level commands *)
let cmds = [
  create_user_cmd; delete_user_cmd; list_users_cmd;
  create_group_cmd; delete_group_cmd; list_groups_cmd;
  create_role_cmd; delete_role_cmd; list_roles_cmd;
]
```

**Good**: Hierarchical grouping with consistent verbs
```ocaml
(* Grouped by resource, consistent verbs *)
let create_cmd = Cmd.v (Cmd.info "create") create_user_term
let delete_cmd = Cmd.v (Cmd.info "delete") delete_user_term
let list_cmd = Cmd.v (Cmd.info "list") list_users_term

let user_cmd =
  let info = Cmd.info "user" ~doc:"Manage users" in
  Cmd.group info ~default:list_users_term [create_cmd; delete_cmd; list_cmd]

let main_cmd =
  let info = Cmd.info "mytool" ~version:"1.0" in
  Cmd.group info [user_cmd; group_cmd; role_cmd]
```

### Error Messages

**Bad**: Unhelpful error that doesn't guide the user
```ocaml
let validate_port p =
  if p < 0 || p > 65535 then `Error (false, "invalid port")
  else `Ok p
```

**Good**: Error explains what's wrong and how to fix it
```ocaml
let validate_port p =
  if p < 0 || p > 65535 then
    `Error (false, Printf.sprintf
      "port %d is out of range (must be 0-65535)" p)
  else `Ok p
```

### Separating Parsing from Logic

**Bad**: Business logic mixed with cmdliner parsing
```ocaml
let run_term =
  let open Term in
  const (fun config_file ->
    (* Business logic embedded in term *)
    let config = read_config config_file in
    let db = connect_db config in
    run_server db)
  $ config_file_arg
```

**Good**: Terms only parse; separate function does the work
```ocaml
(* Pure business logic function *)
let run ~config_file =
  let config = read_config config_file in
  let db = connect_db config in
  run_server db

(* Term just wires up arguments *)
let run_term = Term.(const run $ config_file_arg)
```

### Flag Orthogonality

**Bad**: Flags with hidden interactions
```ocaml
(* --json silently disables --color, user doesn't know *)
let output_format json color =
  if json then Json else if color then Colored else Plain
```

**Good**: Orthogonal flags, explicit conflicts
```ocaml
(* Either format flag, not both *)
let output_format =
  Arg.(value & vflag Plain [
    Json, info ["json"] ~doc:"Output as JSON.";
    Colored, info ["color"] ~doc:"Output with ANSI colors.";
  ])
```

## Cmdliner-Specific Guidance

When writing or revising cmdliner code, follow these patterns:

- Use `Cmd.v` with a `Term.t` and `Cmd.info` for each command or subcommand.
- Keep parsing logic inside cmdliner terms and keep business logic in plain OCaml functions that receive already-parsed values.
- Use `Arg.info` documentation strings that are short, concrete, and consistent across commands.
- Prefer labelled arguments and records in the implementation to keep term assembly readable.
- Ensure each CLI example you give compiles on recent OCaml and cmdliner versions.

### Typical Structure

When the user asks for a new CLI, aim to provide:

1. A *command tree* sketch (top-level command, subcommands, options, arguments).
2. Example `Cmd.t` and `Term.t` definitions.
3. Example `dune` stanzas required to build the executable.
4. Example usage snippets showing common workflows.

## Response Format

Unless the user requests otherwise, structure your responses as:

1. **Overview** – brief description of the CLI design or change.
2. **Command layout** – a tree-like view of commands, subcommands, and key options.
3. **Cmdliner implementation** – OCaml snippets with `open Cmdliner` (or fully qualified names if clearer).
4. **Help and examples** – sample `--help` output and real-world usage examples.
5. **Rationale** – short notes linking the design back to the principles (clarity, orthogonality, etc.).

Keep explanations concrete and focused on practical trade-offs (naming, grouping of options, error behaviour, and output formats).

## CLI Output Design Guidelines

A good CLI is both **useful** and **beautiful**. Follow these guidelines for consistent, professional output.

### Core Libraries

| Library | Purpose |
|---------|---------|
| `fmt` | Styled terminal output (colors, bold, etc.) |
| `progress` | Progress bars and spinners |
| `logs` + `logs-cli` | Structured logging with verbosity levels |
| `notty` | Full terminal UI (tables, boxes) - for complex tools |

### Output Modes

Every CLI should support at least two output modes:

```ocaml
type output_format = Human | Json

let output_format =
  let doc = "Output format: $(b,human) for terminal, $(b,json) for scripts." in
  Arg.(value & opt (enum ["human", Human; "json", Json]) Human &
       info ["o"; "output"] ~doc ~docv:"FORMAT")
```

**Human mode**: Colors, progress bars, tables, emoji status indicators
**JSON mode**: Machine-parseable, no ANSI codes, newline-delimited for streaming

### Color Scheme

Use consistent semantic colors across all tools:

```ocaml
(* Standard semantic styles *)
let success = Fmt.(styled `Green string)      (* ✓ Success, OK *)
let error = Fmt.(styled `Red string)          (* ✗ Error, Failed *)
let warning = Fmt.(styled `Yellow string)     (* ⚠ Warning *)
let info = Fmt.(styled `Cyan string)          (* ℹ Info, hints *)
let dimmed = Fmt.(styled `Faint string)       (* Secondary info *)
let bold = Fmt.(styled `Bold string)          (* Emphasis, headers *)
let code = Fmt.(styled `Cyan string)          (* Code, paths, values *)

(* Status indicators with icons *)
let pp_status ppf = function
  | `Ok -> Fmt.pf ppf "%a" Fmt.(styled `Green string) "✓"
  | `Error -> Fmt.pf ppf "%a" Fmt.(styled `Red string) "✗"
  | `Warning -> Fmt.pf ppf "%a" Fmt.(styled `Yellow string) "⚠"
  | `Info -> Fmt.pf ppf "%a" Fmt.(styled `Cyan string) "ℹ"
  | `Pending -> Fmt.pf ppf "%a" Fmt.(styled `Blue string) "○"
```

### Progress Bars

Use the `progress` library for long-running operations:

```ocaml
open Progress

(* Simple progress bar *)
let with_progress ~total f =
  let bar =
    Line.(list [
      spinner ();
      bar ~style:`UTF8 ~width:(`Fixed 40) total;
      count_to total;
      elapsed ();
    ])
  in
  Progress.with_reporter bar f

(* Example usage *)
let process_files files =
  let total = List.length files in
  with_progress ~total (fun report ->
    List.iteri (fun i file ->
      process_file file;
      report i
    ) files)
```

For indeterminate operations, use spinners:

```ocaml
let with_spinner ~message f =
  let line = Line.(list [spinner (); const message]) in
  Progress.with_reporter line (fun _report -> f ())
```

### Tables

For tabular data, use aligned columns:

```ocaml
(* Simple table with Fmt *)
let pp_table ppf rows =
  let widths = compute_column_widths rows in
  List.iter (fun row ->
    List.iteri (fun i cell ->
      let width = List.nth widths i in
      Fmt.pf ppf "%-*s  " width cell
    ) row;
    Fmt.pf ppf "@."
  ) rows

(* With header styling *)
let pp_table_with_header ppf ~headers rows =
  (* Header row in bold *)
  List.iter (fun h -> Fmt.pf ppf "%a  " bold h) headers;
  Fmt.pf ppf "@.";
  (* Separator *)
  List.iter (fun h -> Fmt.pf ppf "%s  " (String.make (String.length h) '─')) headers;
  Fmt.pf ppf "@.";
  (* Data rows *)
  List.iter (fun row ->
    List.iter (fun cell -> Fmt.pf ppf "%s  " cell) row;
    Fmt.pf ppf "@."
  ) rows
```

### Error Output

Errors should be clear, actionable, and visually distinct:

```ocaml
let pp_error ppf ~context ~message ~hint =
  Fmt.pf ppf "@[<v>%a %a@,%a@,%a %a@]@."
    Fmt.(styled `Red string) "error:"
    Fmt.(styled `Bold string) message
    dimmed (Printf.sprintf "  in %s" context)
    Fmt.(styled `Cyan string) "hint:"
    Fmt.string hint

(* Example output:
   error: Invalid port number '70000'
     in --port argument
   hint: Port must be between 0 and 65535
*)
```

### Summary Output

For commands that process multiple items:

```ocaml
let pp_summary ppf ~processed ~succeeded ~failed ~skipped =
  Fmt.pf ppf "@.%a@."
    Fmt.(styled `Bold string) "Summary:";
  Fmt.pf ppf "  %a %d processed@."
    (Fmt.styled `Cyan string) "•" processed;
  if succeeded > 0 then
    Fmt.pf ppf "  %a %d succeeded@."
      (Fmt.styled `Green string) "✓" succeeded;
  if failed > 0 then
    Fmt.pf ppf "  %a %d failed@."
      (Fmt.styled `Red string) "✗" failed;
  if skipped > 0 then
    Fmt.pf ppf "  %a %d skipped@."
      (Fmt.styled `Yellow string) "○" skipped

(* Example output:
   Summary:
     • 42 processed
     ✓ 40 succeeded
     ✗ 2 failed
*)
```

### TTY Detection

Always check if stdout is a terminal before using colors/progress:

```ocaml
let setup_formatter () =
  let style_renderer =
    if Unix.isatty Unix.stdout then `Ansi_tty else `None
  in
  Fmt.set_style_renderer Fmt.stdout style_renderer

(* Or use Fmt_cli for cmdliner integration *)
let setup_term =
  Term.(const Fmt_tty.setup_std_outputs $ Fmt_cli.style_renderer ())
```

### Verbosity Levels

Integrate with Logs for consistent verbosity:

```ocaml
(* In main.ml *)
let setup_log style_renderer level =
  Fmt_tty.setup_std_outputs ?style_renderer ();
  Logs.set_level level;
  Logs.set_reporter (Logs_fmt.reporter ())

let setup_log_term =
  Term.(const setup_log $ Fmt_cli.style_renderer () $ Logs_cli.level ())

(* In code, use appropriate log levels *)
Logs.debug (fun m -> m "Processing file %s" path);
Logs.info (fun m -> m "Converted %d records" count);
Logs.warn (fun m -> m "Deprecated format, consider upgrading");
Logs.err (fun m -> m "Failed to parse: %s" reason);
```

### Example: Complete CLI with Good Output

```ocaml
open Cmdliner

(* Styled output helpers *)
let success fmt = Fmt.pf Fmt.stdout ("%a " ^^ fmt ^^ "@.")
  Fmt.(styled `Green string) "✓"
let error fmt = Fmt.pf Fmt.stderr ("%a " ^^ fmt ^^ "@.")
  Fmt.(styled `Red string) "✗"
let info fmt = Fmt.pf Fmt.stdout ("%a " ^^ fmt ^^ "@.")
  Fmt.(styled `Cyan string) "ℹ"

(* Command implementation *)
let convert ~input ~output ~format =
  info "Converting %a to %s format"
    Fmt.(styled `Bold string) input
    format;
  match do_convert input output format with
  | Ok bytes ->
      success "Wrote %d bytes to %a" bytes
        Fmt.(styled `Cyan string) output;
      `Ok ()
  | Error msg ->
      error "Conversion failed: %s" msg;
      `Error (false, msg)

(* Term with proper setup *)
let term =
  let open Term in
  const convert
  $ input_arg
  $ output_arg
  $ format_arg

let cmd =
  let info = Cmd.info "convert"
    ~doc:"Convert between formats"
    ~man:[`S "EXAMPLES"; `P "$(iname) input.json -o output.cbor"]
  in
  Cmd.v info Term.(ret (const setup $ setup_log_term $ term))
```

### Checklist for New CLIs

- [ ] Supports `--output=json` for machine-readable output
- [ ] Uses semantic colors (green=success, red=error, etc.)
- [ ] Progress bars for operations > 1 second
- [ ] Clear error messages with hints
- [ ] Summary output for batch operations
- [ ] TTY detection (no colors when piped)
- [ ] Verbosity via `-v` / `--verbosity` (Logs_cli)
- [ ] Consistent with project conventions

