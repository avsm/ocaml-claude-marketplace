---
name: result
description: "OCaml Result type patterns using OCaml 5.x stdlib. Use when Claude needs to: (1) Handle errors with Result types, (2) Chain Result operations with let*, (3) Extract values from Ok/Error, (4) Refactor code using local let* bindings to use Result.Syntax"
---

# OCaml Result Patterns

OCaml 5.x provides `Result.Syntax` for monadic chaining and `Result.get_ok`/`Result.get_error` for extraction.

## Result.Syntax

Use `open Result.Syntax` to get `let*` and `let+` bindings:

```ocaml
open Result.Syntax

let process request =
  let* req = validate request in
  let* auth = authenticate req in
  let* _ = authorize auth in
  execute req
```

**DO NOT** define local `let ( let* ) = Result.bind`. Use `open Result.Syntax` instead.

## Extracting Values

| Function | Behavior on Error |
|----------|-------------------|
| `Result.get_ok r` | Raises `Invalid_argument` |
| `Result.get_error r` | Raises `Invalid_argument` |
| `Result.value r ~default` | Returns default |

Use `Result.get_ok` only when failure is a programming error:

```ocaml
(* Startup/config - crash on failure is intentional *)
let config = Result.get_ok (Config.load ())

(* Test setup - failure means test bug *)
let client = Result.get_ok (Tls.Config.client ~authenticator ())
```

## Custom get_ok

Only define custom `get_ok` when you need different exception behavior:

```ocaml
(* Raises domain-specific Protocol_error instead of Invalid_argument *)
let get_ok = function
  | Ok x -> x
  | Error e -> raise (Protocol_error e)
```

If you just want `Invalid_argument`, use `Result.get_ok` directly.
