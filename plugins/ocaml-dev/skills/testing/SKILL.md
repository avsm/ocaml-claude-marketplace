---
name: testing
description: Testing strategies for OCaml libraries. Use when discussing tests, alcotest, eio mocks, test structure, or test-driven development in OCaml projects.
---

# OCaml Testing

## Test Directory Structure

Use `test/` directory with:
- `test.ml` - Main runner controlling initialization order
- `test_x.ml` - One file per module `x.ml` being tested, exports `suite`

```
lib/
├── foo.ml
└── bar.ml
test/
├── dune
├── test.ml          # Main runner
├── test_foo.ml      # suite : (string * unit Alcotest.test_case list) list
└── test_bar.ml
```

For single-module libraries, a single `test_foo.ml` as runner is acceptable.

## Dune Configuration

```dune
(test
 (name test)
 (libraries mylib alcotest vlog))
```

Add `vlog` for test logging setup with `TEST_LOG` environment variable support.

## Main Runner Pattern (test.ml)

The main `test.ml` controls initialization order for side effects:

```ocaml
(* 1. Initialize RNG before any test module is loaded *)
let () = Crypto_rng_unix.use_default ()

(* 2. Set up logging with Vlog *)
let () = Vlog.setup_test ~level:Logs.Debug ()

(* 3. Run all test suites *)
let () = Alcotest.run "mylib" Test_foo.suite
```

For multiple modules:

```ocaml
let () = Crypto_rng_unix.use_default ()
let () = Alcotest.run "mylib" (Test_foo.suite @ Test_bar.suite)
```

## Module Test File Pattern (test_x.ml)

Each module exports a `suite` value. **Do not initialize RNG or run Alcotest here.**

```ocaml
(** Tests for Foo module. *)

let test_basic () =
  let result = Foo.process "input" in
  Alcotest.(check string) "expected output" "output" result

let test_empty () =
  let result = Foo.process "" in
  Alcotest.(check string) "empty input" "" result

let suite =
  [
    ( "process",
      [
        Alcotest.test_case "basic" `Quick test_basic;
        Alcotest.test_case "empty" `Quick test_empty;
      ] );
  ]
```

## Lazy State for Module-Level Values

If a test module needs RNG at load time, use lazy evaluation:

```ocaml
let key = lazy (Crypto_rng.generate 32)
let key () = Lazy.force key

let test_encrypt () =
  let ciphertext = Foo.encrypt ~key:(key ()) plaintext in
  ...
```

This defers RNG use until tests actually run, after `test.ml` initializes the RNG.

## Alcotest Patterns

### Custom Testables

```ocaml
let result_testable ok_t =
  Alcotest.result ok_t Alcotest.string

let my_type_testable =
  Alcotest.testable My_type.pp My_type.equal
```

### Common Checks

```ocaml
Alcotest.(check int) "count" 42 actual
Alcotest.(check string) "name" expected actual
Alcotest.(check bool) "flag" true actual
Alcotest.(check (list int)) "items" [1;2;3] actual
Alcotest.(check (option string)) "maybe" (Some "x") actual
Alcotest.(check (result int string)) "result" (Ok 42) actual
```

### Testing Exceptions

```ocaml
let test_raises () =
  Alcotest.check_raises "should fail" (Invalid_argument "bad")
    (fun () -> Foo.parse "bad")
```

## Initialization Order

Always initialize in `test.ml` before `Alcotest.run`:

1. **RNG** - `Crypto_rng_unix.use_default ()` or `Mirage_crypto_rng_unix.use_default ()`
2. **Logging** - `Vlog.setup_test ()` (see below)
3. **Other global state** - Environment setup, temp directories

This ensures deterministic test ordering and proper side-effect sequencing.

## Test Logging with `TEST_LOG`

Use `Vlog.setup_test` to configure logging in tests:

```ocaml
let () = Vlog.setup_test ~level:Logs.Debug ()
```

**Default behaviour**: Logs at Debug level. Alcotest captures output to a file by default, so verbose logging doesn't clutter the terminal. Output is shown only when tests fail.

**Override via environment**: Set `TEST_LOG` using RUST_LOG-style syntax:

```bash
# Reduce noise (only warnings/errors)
TEST_LOG=warning dune test

# Per-source overrides
TEST_LOG=warning,conpool:debug dune test

# Multiple source overrides (keep default debug level)
TEST_LOG=http:warning,tls:info dune test
```

Valid levels: `error`, `warning`, `info`, `debug`.

The syntax matches `RUST_LOG`:
- `level` - set global level
- `level,src:level` - global level + per-source override
- `src:level,src:level` - per-source overrides only (keep default)

### Per-Source Overrides in Code

For hardcoded per-source control, set levels after `setup_test`:

```ocaml
let () = Vlog.setup_test ~level:Logs.Debug ()
let () = Logs.Src.set_level Conpool.src (Some Logs.Debug)
let () = Logs.Src.set_level Requests.src (Some Logs.Warning)
```

### Comparison with Other Ecosystems

The `TEST_LOG` pattern follows conventions from other languages:

| Language | Mechanism | Default | Notes |
|----------|-----------|---------|-------|
| **OCaml** | `TEST_LOG` env var | Debug | Alcotest captures output |
| **Rust** | `RUST_LOG` env var | Off | `RUST_LOG=debug` to enable |
| **Go** | `-v` flag + `t.Log` | Buffered | Shown on failure or with `-v` |
| **Python** | pytest capture | Captured | Shown on failure by default |
| **Node.js** | `DEBUG`/`LOG_LEVEL` | Off | `DEBUG=*` to enable |

OCaml + Alcotest is similar to pytest: output is captured by default and shown on failure. This means we can log verbosely without cluttering passing tests.

- [Rust env_logger](https://docs.rs/env_logger/) and [test-log crate](https://github.com/d-e-s-o/test-log) - `RUST_LOG` env var
- [pytest logging](https://pytest-with-eric.com/pytest-best-practices/pytest-logging/) - captures and shows on failure
- [Go testing package](https://pkg.go.dev/testing) - `t.Log` buffers, shown on failure or `-v`
- [Node.js debug module](https://www.npmjs.com/package/debug-level) - `DEBUG` and `DEBUG_LEVEL` env vars

### Why Default to Debug?

1. **Alcotest captures output**: Verbose logs don't clutter terminal
2. **Shown on failure**: When a test fails, you see all the debug info
3. **No re-running needed**: Debug output is already captured
4. **Consistent behaviour**: Same pattern across all monorepo tests

## Core Philosophy

1. **Unit Tests First**: Prioritize unit tests for individual modules and functions.
2. **1:1 Test Coverage**: Every module in `lib/` should have a corresponding test module in `test/`.
3. **Isolated Tests**: Each test should be independent and not rely on external state.
4. **Clear Test Names**: Test names should describe what they test, not how.
5. **Test Inclusion**: All test suites must be included in the main test runner.

## Naming Conventions

- **Test suite names**: lowercase, single words (e.g., `"users"`, `"commands"`, `"process"`)
- **Test case names**: lowercase with underscores, concise but descriptive (e.g., `"basic"`, `"empty_input"`, `"parse_error"`)

## Writing Good Tests

**Function Coverage**: Test all public functions exposed in `.mli` files, including success, error, and edge cases.

**Test Data**: Use helper functions to create test data:

```ocaml
let make_user ?(name = "test") ?(id = 1) () = User.v ~name ~id
```

**Property-Based Testing**: For complex logic, consider property-based testing with QCheck:

```ocaml
let test_roundtrip =
  QCheck.Test.make ~count:1000
    ~name:"encode then decode is identity"
    QCheck.string
    (fun s -> Codec.decode (Codec.encode s) = s)
```

## End-to-End Testing with Cram

Cram tests verify CLI executable behavior.

**Use Cram Directories**: Every Cram test should be a directory ending in `.t` (e.g., `my_feature.t/`).

**Create Actual Test Files**: Avoid embedding code within `run.t` using heredocs. Create real source files within the test directory.

```
test/
└── my_feature.t/
    ├── run.t           # The cram test script
    ├── input.txt       # Test input file
    └── expected.json   # Expected output
```

## Running Tests

```bash
# Run all tests
dune test

# Run tests and watch for changes
dune test -w

# Run a specific test
dune exec test/test.exe -- test "suite_name"

# Run tests with coverage
dune test --instrument-with bisect_ppx
bisect-ppx-report summary
```
