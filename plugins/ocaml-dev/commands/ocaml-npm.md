---
description: Set up npm publishing for OCaml projects via js_of_ocaml/wasm_of_ocaml
argument-hint:
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep]
---

# OCaml NPM Setup

This command sets up the npm publishing workflow for OCaml projects compiled to JavaScript/WASM.

## Arguments

None - operates on the current directory.

## Prerequisites

- Existing OCaml project with dune
- js_of_ocaml or wasm_of_ocaml dependencies configured
- Git repository initialized

## Two-Branch Workflow

```
main branch   - OCaml source code, dune build files, opam packages
npm branch    - Built JavaScript/WASM assets, package.json, README for npm
```

## Process

### 1. Analyze Project

Determine:
- Project name from dune-project
- Existing JS build targets
- Whether to support JS, WASM, or both

### 2. Create/Update Dune Build Rules

Add or update `lib/js/dune` (or appropriate location):

```dune
(library
 (name <project>_js)
 (public_name <project>-js)
 (libraries <project> brr)
 (modes byte)
 (modules <project>_js))

(executable
 (name <project>_js_main)
 (libraries <project>_js)
 (js_of_ocaml)
 (modes js wasm)
 (modules <project>_js_main))

; Friendly filenames
(rule
 (targets <project>.js)
 (deps <project>_js_main.bc.js)
 (action (copy %{deps} %{targets})))

(rule
 (targets <project>.wasm.js)
 (deps <project>_js_main.bc.wasm.js)
 (action (copy %{deps} %{targets})))

; Install rules
(install
 (package <project>-js)
 (section share)
 (files
  <project>.js
  <project>.wasm.js
  (glob_files_rec (<project>_js_main.bc.wasm.assets/* with_prefix <project>_js_main.bc.wasm.assets))))
```

### 3. Update dune-project

Add `-js` package if not present:

```dune
(package
 (name <project>-js)
 (synopsis "Browser library via js_of_ocaml/wasm_of_ocaml")
 (depends
  (ocaml (>= 5.1.0))
  (<project> (= :version))
  (js_of_ocaml (>= 5.0))
  (js_of_ocaml-ppx (>= 5.0))
  (wasm_of_ocaml-compiler (>= 5.0))
  (brr (>= 0.0.6))))
```

### 4. Create npm Branch

Create orphan branch with:
- `package.json`
- `README.md` (browser-focused)
- `LICENSE`
- `release.sh`
- `.gitignore`

### 5. Generate release.sh Script

```bash
#!/bin/bash
# Release script for npm package
set -e

INSTALL_DIR="_build/install/default/share/<project>-js"

# ... copy JS, WASM loader, and assets
```

## Files Created

### On main branch:
- `lib/js/dune` (or updates existing)
- Updates to `dune-project`

### On npm branch (new orphan):
- `package.json`
- `README.md`
- `LICENSE`
- `release.sh`
- `.gitignore`

## Example Usage

```
/ocaml-npm
```

## Success Output

```
Set up npm publishing workflow for: my-library

Main branch updates:
  - Created lib/js/dune with JS/WASM build rules
  - Updated dune-project with my-library-js package

Created npm branch with:
  - package.json (name: my-library-jsoo)
  - README.md (browser usage documentation)
  - LICENSE
  - release.sh (copies built assets)
  - .gitignore

Workflow:
  1. Develop on main branch
  2. Build: opam exec -- dune build @install
  3. Switch to npm: git checkout npm
  4. Copy assets: ./release.sh
  5. Publish: npm publish

Current branch: main
To switch to npm branch: git checkout npm
```
