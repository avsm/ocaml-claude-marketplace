---
name: ocaml
description: "OCaml development guidance for building robust, type-safe applications. Use when Claude needs to: (1) Write OCaml code following modern best practices, (2) Design module interfaces (.mli files), (3) Handle errors with result types, (4) Work with dune build system, (5) Use common OCaml libraries (eio, fmt, logs, cmdliner, yojson, cohttp-eio), or any other OCaml development tasks"
---

# OCaml Development

## Core Philosophy

1. **Interface-First Design**: Design the `.mli` file first. A clean interface matters more than clever implementation.
2. **Modularity**: Build small, focused modules that do one thing well. Compose them for larger systems.
3. **Simplicity (KISS)**: Prioritize clarity over conciseness. Avoid obscure constructs.
4. **Explicitness**: Make control flow and error handling explicit. Avoid exceptions for recoverable errors.
5. **NEVER use Obj.magic**: It breaks type safety. There is always a better solution.

## Build System and Tooling

- **Build**: Use `dune` exclusively
- **Formatting**: Run `dune fmt` before committing (uses ocamlformat)
- **Common Libraries**:
  - Concurrency: `eio`
  - Structured output: `fmt`
  - Logging: `logs`
  - CLI parsing: `cmdliner`
  - JSON: `yojson`
  - HTTP: `cohttp-eio`

## Module Interface Design

### Documentation Pattern

Every `.mli` file starts with a top-level doc comment:

```ocaml
(** User API

    This module provides types and functions for interacting with users. *)
```

### Function Documentation

Use `[function_name arg1 arg2] is ...` pattern:

```ocaml
val is_bot : t -> bool
(** [is_bot u] is [true] if [u] is a bot user. *)
```

For values, describe what they represent:

```ocaml
type id = string
(** A user identifier. *)
```

### Standard Interface for Data Types

For modules with a central type `t`, provide these functions where applicable:

| Function | Purpose |
|----------|---------|
| `val v : ... -> t` | Pure smart constructor (no I/O) |
| `val create : ... -> (t, Error.t) result` | Constructor with side-effects |
| `val pp : t Fmt.t` | Pretty-printer for logging/debugging |
| `val equal : t -> t -> bool` | Structural equality |
| `val compare : t -> t -> int` | Comparison for sorting |
| `val of_json : Yojson.Safe.t -> (t, string) result` | Parse from JSON |
| `val to_json : t -> Yojson.Safe.t` | Serialize to JSON |
| `val validate : t -> (t, string) result` | Validate data integrity |

### Abstract Types

Keep types abstract (`type t`) when possible. Expose smart constructors and accessors instead of record fields to maintain invariants.

## Error Handling

Use `result` type for recoverable errors. Reserve exceptions for programming errors (e.g., `Invalid_argument`).

### Central Error Type

Define a comprehensive error type in `lib/error.ml`:

```ocaml
(* In lib/error.mli *)
type t = [
  | `Api of string * Yojson.Safe.t
  | `Json_parse of string
  | `Network of string
  | `Msg of string
]

val pp : t Fmt.t
```

### Error Helper Pattern

```ocaml
let err_api code fields = Error (`Api (code, fields))
let err_parse msg = Error (`Json_parse msg)

let find_user_id json =
  match Yojson.Safe.Util.find_opt "id" json with
  | Some (`String id) -> Ok id
  | Some _ -> err_parse "Expected string for user ID"
  | None -> err_parse "Missing user ID"
```

### Rules

- **Never** use `try ... with _ -> ...`. Match specific exceptions.
- For unrecoverable startup errors, use `Fmt.failwith`:

```ocaml
let tls_config =
  match Tls.Config.client ~authenticator () with
  | Ok config -> config
  | Error (`Msg msg) -> Fmt.failwith "Failed to create TLS config: %s" msg
```

## Function Design

- **Keep functions small**: One function, one purpose. Decompose complex logic.
- **Avoid deep nesting**: More than 2-3 levels of `match`/`if` signals need for refactoring.
- **Prefer purity**: Isolate side-effects at edges (`bin/`, `lib/ui/`).
- **Composition over abstraction**: Favor small concrete functions over deep abstractions.
- **Data-oriented**: Operate on simple, immutable data structures.
- **No premature generalization**: Solve the problem at hand, avoid unnecessary complexity.

## Logging

Use the `logs` library with per-module log sources:

```ocaml
let log_src = Logs.Src.create "project_name.module_name"
module Log = (val Logs.src_log log_src : Logs.LOG)
```

### Log Levels

| Level | Use Case |
|-------|----------|
| `Log.app` | Messages always shown to user (startup) |
| `Log.err` | Handled but critical errors |
| `Log.warn` | Potential issues, operation continues |
| `Log.info` | Informational state messages |
| `Log.debug` | Verbose debugging details |

### Structured Logging

```ocaml
Log.info (fun m ->
    m "Received event: %s" event_type
      ~tags:(Logs.Tag.add "channel_id" channel_id Logs.Tag.empty))
```

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Files | lowercase_underscores | `user_profile.ml` |
| Modules | lowercase_underscores | `user_profile` |
| Primary type | `t` | `type t` |
| Identifiers | `id` | `type id = string` |
| Values | short_descriptive | `find_user`, `create_channel` |

### Labels

Use labels only when they clarify meaning. Avoid `~f` and `~x`.

## CLI Applications

For `bin/` applications using `cmdliner`:

- Place shared functionality (auth, logging setup) in `bin/common.ml`
- Provide a shared `run` function that initializes the main loop and environment (e.g., Eio loop)
- All commands should use this function for consistent environment

## Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

**Format**: `type(scope): subject`

| Type | Purpose |
|------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation |
| `style` | Formatting |
| `refactor` | Code restructuring |
| `test` | Tests |
| `chore` | Maintenance |

**Examples**:
- `feat(api): add support for file uploads`
- `fix(ui): correct channel list rendering bug`
- `test(user): add tests for user profile updates`
