# OCaml Development Plugin

Comprehensive OCaml development toolkit for Claude Code.

## Features

### Slash Commands

| Command | Description |
|---------|-------------|
| `/init-ocaml [name]` | Initialize a new OCaml project |
| `/port-to-dune` | Migrate from ocamlbuild/topkg to dune |
| `/add-rfc <number>` | Fetch RFC and add OCamldoc citations |
| `/ocaml-npm` | Set up npm publishing workflow |
| `/tidy [path]` | Refactor OCaml code for style |

### Skills (Auto-invoked)

| Skill | Triggers On |
|-------|-------------|
| ocaml-project-setup | "new project", "init", "dune-project" |
| ocaml-testing | "test", "alcotest", "eio mock" |
| ocaml-rfc-integration | "RFC", "specification", "standard" |
| ocaml-dune-migration | "ocamlbuild", "topkg", "_tags" |
| ocaml-npm-publishing | "npm", "js_of_ocaml", "browser" |
| ocaml-code-style | "tidy", "refactor", "clean up" |
| ocaml-tutorials | "tutorial", ".mld", "MDX" |

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
  "copyright_year_start": 2025,
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
