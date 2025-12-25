---
description: Fetch an IETF RFC and integrate it into the project with OCamldoc citations
argument-hint: <rfc-number>
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, WebFetch]
---

# Add RFC

This command fetches an IETF RFC specification and integrates it into the project.

## Arguments

RFC number: $ARGUMENTS (required, e.g., "6265" for HTTP cookies)

## Process

### 1. Fetch the RFC

Fetch from IETF in plain text format:
```
https://datatracker.ietf.org/doc/html/rfc<NUMBER>.txt
```

**Important**: Always use `.txt` extension, not `.html`.

### 2. Save to spec/ Directory

Create `spec/` directory if it doesn't exist and save:
```
spec/rfc<NUMBER>.txt
```

### 3. Parse RFC Structure

Extract key information:
- RFC title and abstract
- Table of contents (sections)
- Key terminology definitions
- Related RFCs (obsoletes, updates, references)

### 4. Generate OCamldoc Citation Templates

Provide ready-to-use citations for:

**Module-level:**
```ocaml
(** RFC <NUMBER> <Title>.

    This module implements <description> as specified in
    {{:https://datatracker.ietf.org/doc/html/rfc<NUMBER>}RFC <NUMBER>}.

    {2 References}
    {ul
    {- {{:https://datatracker.ietf.org/doc/html/rfc<NUMBER>}RFC <NUMBER>} - <Title>}} *)
```

**Section-specific:**
```ocaml
(** Implements {{:https://datatracker.ietf.org/doc/html/rfc<NUMBER>#section-N}RFC <NUMBER> Section N}. *)
```

### 5. Suggest Integration Points

Search the codebase for:
- Functions that might implement RFC sections
- Existing RFC references that could be improved
- Types that represent RFC concepts

## Output

Provide:
1. Confirmation of RFC download
2. Summary of RFC contents (title, sections)
3. OCamldoc citation templates
4. Suggestions for where to add citations in existing code

## Example Usage

```
/add-rfc 6265
/add-rfc 3492
```

## Success Output

```
Fetched RFC 6265: HTTP State Management Mechanism

Saved to: spec/rfc6265.txt

RFC Summary:
  Title: HTTP State Management Mechanism
  Obsoletes: RFC 2965
  Sections:
    1. Introduction
    2. Conventions
    3. Overview
    4. Server Requirements
    5. User Agent Requirements
    6. Implementation Considerations
    7. Privacy Considerations
    8. Security Considerations

OCamldoc Templates:

Module-level:
(** RFC 6265 HTTP State Management Mechanism.

    This module implements HTTP cookie handling as specified in
    {{:https://datatracker.ietf.org/doc/html/rfc6265}RFC 6265}. *)

Section references:
- {{:https://datatracker.ietf.org/doc/html/rfc6265#section-4}RFC 6265 Section 4} - Server Requirements
- {{:https://datatracker.ietf.org/doc/html/rfc6265#section-5}RFC 6265 Section 5} - User Agent Requirements
- {{:https://datatracker.ietf.org/doc/html/rfc6265#section-5.2}RFC 6265 Section 5.2} - The Set-Cookie Header

Related RFCs you might also need:
- RFC 2616 - HTTP/1.1 (obsoleted by RFC 7230-7235)
- RFC 7230 - HTTP/1.1 Message Syntax and Routing
```
