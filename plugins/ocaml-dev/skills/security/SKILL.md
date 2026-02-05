---
name: security
description: "Security hardening for OCaml libraries through systematic vulnerability research. Use when Claude needs to: (1) Research CVEs in similar implementations (C, Rust, Go, Python) and add regression tests, (2) Add fuzz tests for parsers and encoders, (3) Audit integer handling and buffer operations, (4) Test boundary conditions and malformed input, (5) Review cryptographic usage, (6) Add defensive checks against common vulnerability classes"
---

# OCaml Security Audit

Systematic security hardening through vulnerability research, defensive coding, and comprehensive testing.

## Core Philosophy

1. **Study the attacks first**: Research CVEs in equivalent C/Rust/Go/Python libraries before writing tests
2. **Assume hostile input**: Every parser, decoder, and protocol handler receives adversarial data
3. **Fail explicitly**: Reject malformed input early with clear errors, never silently corrupt
4. **Test the boundaries**: Edge cases at min/max values, empty input, and overflow points
5. **Defense in depth**: Multiple validation layers, even when one seems sufficient

## Workflow

### Phase 1: CVE Research

Before writing any tests, research known vulnerabilities in equivalent implementations.

**1. Identify comparable libraries:**

```
OCaml library     → Research in
─────────────────────────────────────
PNG/image parser  → libpng, image-rs, Pillow
TLS/crypto        → OpenSSL, BoringSSL, rustls
HTTP parser       → http-parser, hyper, httptools
JSON parser       → json-c, serde_json, ujson
XML parser        → libxml2, quick-xml, defusedxml
Archive (zip/tar) → libarchive, zip-rs, tarfile
DNS resolver      → c-ares, trust-dns, dnspython
Compression       → zlib, miniz, flate2
YAML parser       → libyaml, serde_yaml, PyYAML
PDF parser        → poppler, pdf-rs, PyPDF2
ASN.1/X.509       → OpenSSL, ring, pyasn1
```

**2. Search for CVEs:**

```bash
# Search NVD database
curl "https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=libpng+buffer+overflow" | jq '.vulnerabilities[].cve.descriptions[0].value'

# Search CVE Details
# https://www.cvedetails.com/vulnerability-search.php

# Search GitHub Security Advisories
gh api graphql -f query='{ securityAdvisories(first:10, ecosystem:PIP, keyword:"pillow") { nodes { summary description severity } } }'

# Check OSV database
curl "https://api.osv.dev/v1/query" -d '{"package": {"name": "pillow", "ecosystem": "PyPI"}}'
```

**3. Document findings** - track CVEs and map to tests. See `references/vulnerability-classes.md`.

### Phase 2: Vulnerability Classes

For each vulnerability class, add targeted tests.

#### Integer Handling

```ocaml
(** Test integer overflow in length calculations.
    CVE pattern: libpng, zlib, many image libraries *)
let test_length_overflow () =
  let huge_width = Int.max_int in
  let huge_height = 1 in
  match Image.create ~width:huge_width ~height:huge_height with
  | Error `Overflow -> ()
  | Error _ -> Alcotest.fail "wrong error type"
  | Ok _ -> Alcotest.fail "should reject overflow"

(** Test signed/unsigned confusion.
    Signed length interpreted as huge unsigned value. *)
let test_negative_length () =
  let data = Bytes.create 4 in
  Bytes.set_int32_be data 0 (-1l);  (* 0xFFFFFFFF *)
  match Parser.read_with_length data with
  | Error (`Invalid_length _) -> ()
  | _ -> Alcotest.fail "should reject negative length"
```

#### Buffer Boundaries

```ocaml
(** Test out-of-bounds read.
    Claimed length exceeds actual data. *)
let test_oob_read () =
  let header = "\x00\x00\x00\x10" in  (* Claims 16 bytes *)
  let data = header ^ "short" in       (* Only 5 bytes of payload *)
  match Parser.decode data with
  | Error (`Truncated _) -> ()
  | Error _ -> Alcotest.fail "wrong error"
  | Ok _ -> Alcotest.fail "should reject truncated data"

(** Test empty input handling. *)
let test_empty_input () =
  match Parser.decode "" with
  | Error _ -> ()  (* Any error is acceptable *)
  | Ok _ -> Alcotest.fail "should reject empty input"
```

#### Denial of Service

```ocaml
(** Test resource exhaustion - deeply nested structures.
    CVE pattern: ujson, many JSON parsers *)
let test_deep_nesting () =
  let depth = 10000 in
  let nested = String.concat "" (List.init depth (fun _ -> "[")) ^
               String.concat "" (List.init depth (fun _ -> "]")) in
  match Json.parse nested with
  | Error (`Nesting_too_deep _) -> ()
  | Error _ -> ()  (* Resource errors are acceptable *)
  | Ok _ -> Alcotest.fail "should limit nesting depth"

(** Test exponential expansion (billion laughs).
    CVE pattern: XML entity expansion *)
let test_entity_expansion () =
  let malicious = "<!DOCTYPE x [<!ENTITY a \"aaa...(1000 chars)\">]><x>&a;&a;&a;...</x>" in
  match Xml.parse malicious with
  | Error _ -> ()
  | Ok _ -> Alcotest.fail "should reject entity expansion"
```

### Phase 3: Fuzz Testing

Add fuzz tests for all parsers and encoders. See the **fuzz** skill for patterns.

**Priority targets:**
1. Binary protocol parsers (highest risk)
2. Text format parsers (JSON, XML, config files)
3. Cryptographic operations
4. Compression/decompression
5. Character encoding conversions

```ocaml
(** Fuzz test: parser must not crash on any input. *)
let test_decode_crash_safety buf =
  let buf = truncate buf in
  let _ = Parser.decode (to_bytes buf) in
  ()

(** Fuzz test: encoder output must be parseable. *)
let test_roundtrip buf =
  let buf = truncate buf in
  match Parser.decode (to_bytes buf) with
  | Error _ -> ()
  | Ok v ->
      let encoded = Parser.encode v in
      match Parser.decode encoded with
      | Error _ -> fail "encoded data failed to parse"
      | Ok v' -> if v <> v' then fail "roundtrip mismatch"
```

### Phase 4: CVE Regression Tests

For each applicable CVE, write a targeted regression test.

```ocaml
(** CVE-2023-XXXX regression test.

    Reference: https://nvd.nist.gov/vuln/detail/CVE-2023-XXXX

    Integer overflow when calculating buffer size from untrusted
    width/height values. *)
let test_cve_2023_xxxx () =
  let malicious_input = Bytes.of_string "\xff\xff\xff\xff\x00\x00\x00\x01" in
  match Image.decode malicious_input with
  | Error _ -> ()
  | Ok _ -> Alcotest.fail "CVE-2023-XXXX: should reject overflow"
```

## Defensive Coding Patterns

### Validate Early, Fail Fast

```ocaml
let decode buf =
  (* Check minimum size before any parsing *)
  if Bytes.length buf < header_size then
    Error (`Truncated { expected = header_size; got = Bytes.length buf })
  else
    let length = Bytes.get_int32_be buf 0 |> Int32.to_int in
    (* Validate length before allocating *)
    if length < 0 then
      Error (`Invalid_length length)
    else if length > max_message_size then
      Error (`Message_too_large { size = length; max = max_message_size })
    else if Bytes.length buf < header_size + length then
      Error (`Truncated { expected = header_size + length; got = Bytes.length buf })
    else
      parse_body buf length
```

### Safe Integer Arithmetic (Hacker's Delight style)

Overflow detection without branches where possible, using bit manipulation.

```ocaml
(** Detect signed addition overflow (Hacker's Delight, 2-13).
    Overflow iff both operands have same sign and result has different sign. *)
let add_overflow a b =
  let sum = a + b in
  (* (a ^ sum) & (b ^ sum) is negative iff overflow *)
  (a lxor sum) land (b lxor sum) < 0

(** Detect signed multiplication overflow (Hacker's Delight, 2-13).
    For non-negative operands: a * b overflows iff a > max_int / b *)
let mul_overflow a b =
  if b = 0 then false
  else if b = -1 then a = Int.min_int  (* Special case: min_int * -1 *)
  else if b > 0 then a > Int.max_int / b || a < Int.min_int / b
  else (* b < -1 *) a > Int.min_int / b || a < Int.max_int / b

(** Safe addition with overflow check. *)
let safe_add a b =
  if add_overflow a b then Error `Overflow
  else Ok (a + b)

(** Safe multiplication with overflow check. *)
let safe_mul a b =
  if mul_overflow a b then Error `Overflow
  else Ok (a * b)

(** Unsigned comparison for signed integers (Hacker's Delight, 2-12).
    Interprets both values as unsigned. *)
let unsigned_lt a b =
  (a lxor Int.min_int) < (b lxor Int.min_int)

(** Check if value fits in n bits unsigned. *)
let fits_in_bits n v =
  v >= 0 && v < (1 lsl n)

(** Use in size calculations. *)
let calculate_buffer_size ~width ~height ~bytes_per_pixel =
  let open Result.Syntax in
  let* row_size = safe_mul width bytes_per_pixel in
  let* total = safe_mul row_size height in
  if total > max_buffer_size then Error `Buffer_too_large
  else Ok total
```

### Safe Integer Narrowing

```ocaml
(** Safe conversion from int to int32. *)
let int_to_int32 n =
  if n < Int32.(to_int min_int) || n > Int32.(to_int max_int) then
    Error `Overflow
  else
    Ok (Int32.of_int n)

(** Safe conversion from int64 to int. *)
let int64_to_int n =
  if n < Int64.of_int Int.min_int || n > Int64.of_int Int.max_int then
    Error `Overflow
  else
    Ok (Int64.to_int n)

(** Convert length field to int, rejecting values that don't fit. *)
let length_field_to_int len_field =
  let open Result.Syntax in
  let* n = int64_to_int len_field in
  if n < 0 then Error (`Invalid_length n)
  else Ok n
```

### Constant-Time Comparison

```ocaml
(** Constant-time string comparison for secrets.
    Prevents timing side-channels when comparing MACs, tokens, etc. *)
let constant_time_equal a b =
  let len_a = String.length a in
  let len_b = String.length b in
  let result = ref (len_a lxor len_b) in
  for i = 0 to min len_a len_b - 1 do
    result := !result lor (Char.code a.[i] lxor Char.code b.[i])
  done;
  !result = 0
```

### Resource Limits

```ocaml
type limits = {
  max_depth : int;
  max_string_length : int;
  max_array_length : int;
  max_total_size : int;
}

let default_limits = {
  max_depth = 100;
  max_string_length = 10_000_000;
  max_array_length = 100_000;
  max_total_size = 100_000_000;
}

let parse ?(limits = default_limits) input =
  parse_with_limits ~limits input
```

## Security Checklist

Before releasing any parser, encoder, or protocol handler:

### Input Validation
- [ ] Empty input rejected or handled correctly
- [ ] Minimum size checked before parsing
- [ ] Maximum size limits enforced
- [ ] Length fields validated before use
- [ ] Negative lengths rejected

### Integer Safety
- [ ] Multiplication overflow checked in size calculations
- [ ] Addition overflow checked in offset calculations
- [ ] No signed/unsigned confusion in lengths
- [ ] Cast results checked when narrowing

### Resource Limits
- [ ] Nesting depth limited
- [ ] Collection sizes limited
- [ ] Total memory usage bounded
- [ ] Recursion depth bounded

### Fuzz Testing
- [ ] Crash-safety test with arbitrary bytes
- [ ] Roundtrip test for encode/decode pairs
- [ ] AFL campaign run for 24+ hours
- [ ] No crashes or hangs found

### CVE Coverage
- [ ] CVEs in similar libraries researched
- [ ] Regression tests written for applicable CVEs
- [ ] Tests documented with CVE references

## References

- `references/vulnerability-classes.md` - Detailed patterns for each vulnerability class
- **fuzz** skill - Comprehensive fuzz testing patterns
- **ocaml-testing** skill - Unit test organization
