# OCaml Claude Marketplace

A collection of Claude Code plugins for OCaml development, based initially on
the experiences in <https://anil.recoil.org/notes/2025-aoah>

## Installation

Add this marketplace to Claude Code:

```
/install-marketplace avsm/ocaml-claude-marketplace
```

Then install the OCaml development plugin:

```
/install ocaml-dev
```

## Available Plugins

### ocaml-dev

Comprehensive OCaml development toolkit including:

**Slash Commands:**
- `/init-ocaml [name]` - Initialize a new OCaml project with dune, opam, CI, and standard files
- `/port-to-dune` - Migrate ocamlbuild/topkg projects to dune
- `/add-rfc <number>` - Fetch an IETF RFC and integrate with ocamldoc citations
- `/ocaml-npm` - Set up npm publishing for js_of_ocaml/wasm_of_ocaml projects
- `/tidy` - Refactor OCaml code to be more idiomatic

**Skills (auto-invoked):**
- Project setup and metadata standards
- Testing strategies (including Eio mocks)
- RFC integration and documentation
- Build system migration
- NPM/browser publishing workflow
- Code style and refactoring patterns
- Tutorial creation with MDX

**LSP Integration:**
- ocamllsp for `.ml`, `.mli`, `.mly`, `.mll` files

## User Configuration

Create `~/.claude/ocaml-config.json` for personalized settings:

```json
{
  "author": {
    "name": "Your Name",
    "email": "you@example.com"
  },
  "license": "ISC",
  "ci_platform": "github",
  "git_hosting": {
    "type": "github",
    "org": "your-username"
  },
  "ocaml_version": "5.2.0"
}
```

If not configured, commands will prompt for required values.

## CI Platform Support

The plugin includes CI templates for:
- GitHub Actions
- Tangled.org
- GitLab CI

Select your preferred platform in the configuration.

## License

ISC License
