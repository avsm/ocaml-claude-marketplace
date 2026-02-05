---
name: logs
description: "OCaml Logs library patterns for structured logging. Use when Claude needs to: (1) Add logging to OCaml modules, (2) Create per-module log sources, (3) Use appropriate log levels, (4) Add structured tags to log messages"
---

# OCaml Logs Patterns

The `logs` library provides structured logging with per-module sources.

## Module Setup

Every module that logs should create its own source:

```ocaml
let log_src = Logs.Src.create "project.module_name"
module Log = (val Logs.src_log log_src : Logs.LOG)
```

This is the standard idiom. Do NOT attempt to abstract or deduplicate this boilerplate.

## Log Levels

| Level | Function | Use Case |
|-------|----------|----------|
| App | `Log.app` | Always shown to user (startup, completion) |
| Error | `Log.err` | Handled but critical errors |
| Warning | `Log.warn` | Potential issues, operation continues |
| Info | `Log.info` | State transitions, progress |
| Debug | `Log.debug` | Verbose debugging details |

## Basic Logging

```ocaml
Log.info (fun m -> m "Processing %d items" count);
Log.err (fun m -> m "Failed to connect: %s" reason);
Log.debug (fun m -> m "Request: %a" Request.pp req)
```

## Structured Tags

Add context with tags for filtering/analysis:

```ocaml
Log.info (fun m ->
    m "Event received: %s" event_type
      ~tags:(Logs.Tag.add "channel_id" channel Logs.Tag.empty))
```

## Reporter Setup (bin/)

In CLI applications, set up the reporter in `bin/common.ml`:

```ocaml
let setup_log style_renderer level =
  Fmt_tty.setup_std_outputs ?style_renderer ();
  Logs.set_level level;
  Logs.set_reporter (Logs_fmt.reporter ())

open Cmdliner

let setup_log =
  Term.(const setup_log
        $ Fmt_cli.style_renderer ()
        $ Logs_cli.level ())
```
