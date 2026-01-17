# OCaml Development Plugin

Comprehensive OCaml development toolkit for Claude Code.

## Features

### Slash Commands

| Command | Description |
|---------|-------------|
| `/init-ocaml [name]` | Initialize a new OCaml project with dune, opam, CI, and standard files |
| `/port-to-dune` | Migrate from ocamlbuild/topkg to dune build system |
| `/add-rfc <number>` | Fetch IETF RFC and add OCamldoc citations |
| `/ocaml-npm` | Set up npm publishing workflow for js_of_ocaml/wasm_of_ocaml |
| `/tidy [path]` | Refactor OCaml code to be more idiomatic and maintainable |

### Skills (Auto-invoked)

#### Project & Build

| Skill | Description |
|-------|-------------|
| ocaml-project-setup | Project structure, dune-project, .mli files, CI configuration |
| ocaml-dune-migration | Migrating from ocamlbuild/topkg to dune (_tags, .mllib, pkg.ml) |
| ocaml-npm-publishing | Publishing to npm via js_of_ocaml and wasm_of_ocaml |

#### Code Quality

| Skill | Description |
|-------|-------------|
| ocaml-code-style | Refactoring patterns, naming conventions, module hygiene |
| ocaml-testing | Testing with Alcotest, Eio mocks, cram tests |
| fuzz | Fuzz testing with Crowbar for parsers and encoders |

#### Documentation

| Skill | Description |
|-------|-------------|
| ocaml-docs | Fixing odoc warnings, reference syntax, cross-package refs |
| ocaml-tutorials | Creating .mld tutorials with MDX executable examples |
| ocaml-rfc-integration | Working with IETF RFCs, OCamldoc citations |

#### Libraries & Frameworks

| Skill | Description |
|-------|-------------|
| eio | Eio concurrency: fibers, switches, mocks, cohttp-eio, bytesrw |
| cmdliner | CLI design with cmdliner following Bünzli's principles |
| jsont | Type-safe JSON encoding/decoding with jsont |
| effects | OCaml 5 algebraic effects design patterns |

#### Performance & Advanced

| Skill | Description |
|-------|-------------|
| memtrace | Allocation profiling to identify hotspots |
| oxcaml | OxCaml extensions: modes, stack allocation, unboxed types, SIMD, zero-alloc |

### LSP Integration

Includes ocamllsp configuration for enhanced code intelligence:
- `.ml` - OCaml source
- `.mli` - OCaml interface
- `.mly` - Menhir grammar
- `.mll` - OCamllex lexer

## Configuration

User settings are read from `~/.claude/ocaml-config.json`:

```json
{
  "author": {
    "name": "Your Name",
    "email": "you@example.com"
  },
  "license": "ISC",
  "copyright_year_start": 2026,
  "ci_platform": "github",
  "git_hosting": {
    "type": "github",
    "org": "username"
  },
  "opam_overlay": {
    "enabled": false,
    "path": null,
    "name": null
  },
  "ocaml_version": "5.2.0"
}
```

### Configuration Options

| Field | Description | Values |
|-------|-------------|--------|
| `license` | Default license for new projects | `ISC`, `MIT`, `Apache-2.0` |
| `ci_platform` | CI system for new projects | `github`, `tangled`, `gitlab` |
| `git_hosting.type` | Git hosting provider | `github`, `tangled`, `gitlab` |
| `ocaml_version` | Minimum OCaml version | e.g., `5.2.0` |

## Usage Examples

### Create a New Project

```
/init-ocaml my-library
```

Creates:
- dune-project with opam generation
- Standard dune files
- .ocamlformat, .gitignore
- LICENSE.md, README.md
- CI configuration
- lib/ and test/ directories

### Migrate from ocamlbuild

```
/port-to-dune
```

Analyzes _tags, .mllib, pkg/pkg.ml and generates dune equivalents.

### Add RFC Documentation

```
/add-rfc 6265
```

Fetches RFC 6265 (HTTP cookies) to spec/, provides OCamldoc citation templates.

### Set Up NPM Publishing

```
/ocaml-npm
```

Creates npm branch workflow for js_of_ocaml/wasm_of_ocaml output.

### Refactor Code

```
/tidy lib/parser.ml
```

Analyzes and suggests idiomatic OCaml improvements.

## Template Files

Templates are in `skills/*/templates/`:

- CI configurations (GitHub, Tangled, GitLab)
- dune-project and dune file templates
- License files (ISC, MIT)
- Test templates (basic, Eio mock)
- npm publishing templates

## License

ISC License
