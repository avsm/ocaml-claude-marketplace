---
description: Refactor and tidy OCaml code to be more idiomatic and maintainable
argument-hint: [file-or-directory]
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep]
---

# Tidy OCaml Code

This command analyzes and refactors OCaml code to make it more idiomatic, maintainable, and concise.

## Arguments

Optional file or directory path: $ARGUMENTS (defaults to current directory)

## Analysis Categories

### 1. Option and Result Patterns

**Look for:**
- Verbose match expressions that could use combinators
- Nested Option/Result handling without let*/let+
- Direct pattern matching where `Option.map`, `Option.bind`, etc. would be clearer

**Transform:**
```ocaml
(* Before *)
match get_value () with
| Some x -> Some (x + 1)
| None -> None

(* After *)
Option.map (fun x -> x + 1) (get_value ())
```

### 2. Monadic Syntax

**Look for:**
- Deeply nested match expressions on Result/Option
- Manual error propagation chains

**Transform:**
```ocaml
(* Before *)
match fetch_user id with
| Ok user ->
    (match fetch_perms user with
     | Ok perms -> Ok (user, perms)
     | Error e -> Error e)
| Error e -> Error e

(* After *)
let open Result.Syntax in
let* user = fetch_user id in
let+ perms = fetch_perms user in
(user, perms)
```

### 3. Pattern Matching vs Conditionals

**Look for:**
- Nested if/then/else chains
- Boolean condition checking that could be pattern matching

**Transform:**
```ocaml
(* Before *)
if x > 0 then
  if x < 10 then "small"
  else "large"
else "negative"

(* After *)
match x with
| x when x < 0 -> "negative"
| x when x < 10 -> "small"
| _ -> "large"
```

### 4. Code Duplication

**Look for:**
- Repeated error message patterns
- Similar function implementations
- Copy-pasted code blocks

**Suggest:**
- Helper functions
- Parameterized abstractions
- Shared error constructors

### 5. Module Hygiene

**Look for:**
- Generic module names (Util, Helpers, Common)
- Exposed record types without abstract `type t`
- Missing pretty-printers for main types
- Unlabeled boolean parameters

### 6. Modern OCaml Patterns

**Suggest:**
- Labeled arguments for clarity
- Local opens for syntax extensions
- Explicit type annotations on public interfaces

## Workflow

1. If a specific file is given, analyze that file
2. Otherwise, find all `.ml` and `.mli` files
3. Analyze each file for improvement opportunities
4. Prioritize by impact:
   - Code duplication (highest)
   - Verbose monadic code
   - Module structure issues
   - Nested conditionals
5. Present findings with before/after examples
6. Apply changes with user confirmation

## Output Format

```
Analysis of lib/parser.ml:

1. OPTION COMBINATORS (3 occurrences)
   Line 45: Replace match with Option.map
   Line 89: Replace match with Option.bind
   Line 123: Replace match with Option.value

2. MONADIC SYNTAX (1 occurrence)
   Lines 156-178: Use let*/let+ for cleaner Result chaining

   Before:
   [code snippet]

   After:
   [refactored snippet]

3. CODE DUPLICATION (2 occurrences)
   Lines 34, 67: Duplicate error construction
   Suggestion: Create helper function `parse_error`

Apply changes? [y/n/select]
```

## Example Usage

```
/tidy
/tidy lib/parser.ml
/tidy src/
```

## Success Output

```
Analyzed 12 OCaml files in lib/

Improvements found:
  - 8 Option/Result patterns to simplify
  - 3 monadic chains to use let*/let+
  - 2 code duplication instances
  - 1 module naming issue

Applied 13 changes:
  - lib/parser.ml: 5 changes
  - lib/types.ml: 3 changes
  - lib/utils.ml: 5 changes (renamed to lib/string_ext.ml)

Reduced total lines: 847 -> 762 (-10%)

Suggestions not applied (require manual review):
  - Consider abstracting User.t type (lib/user.ml:12)
  - Add pp function to Config module (lib/config.ml)
```
