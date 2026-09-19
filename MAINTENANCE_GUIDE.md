# Cloudflare Turnstile Solver — Authorized Maintenance and Traffic-Analysis Guide



## 1. Project status

This repository is a Rust, request-oriented research implementation. The upstream README explicitly says that it is outdated and no longer works. It contains no WAF implementation, and a Turnstile token is not equivalent to WAF approval: Cloudflare evaluates the complete request, session, client, and site configuration independently.

The maintenance objective should therefore be **reproducible analysis and compatibility testing in a controlled lab**, not a promise that arbitrary third-party challenges can be solved.

## 2. Repository architecture

```text
src/
  deobfuscator/       Oxc-based JavaScript AST transforms
  parser/             Payload, VM, offsets, magic-bit, and function extraction
  disassembler/       Bytecode/instruction decoding
  decompiler/         Research/debugging decompiler components
  reverse/            Compression and response/payload transformation code
  solver/             Task orchestration, HTTP client, fingerprint model, VM parser
  bin/solve_test/     Local executable used by the original author for testing
```

The public crate surface is declared in `src/lib.rs`. `TurnstileSolver` in `src/solver/mod.rs` loads fingerprint fixtures from `workspace/cloudflare_test.json`, creates a `TurnstileTask`, and `task.rs` coordinates challenge retrieval, JavaScript analysis, payload construction, and response processing.

### JavaScript path

`src/deobfuscator/mod.rs` parses non-module JavaScript with Oxc and applies these visitors in order:

1. `numbers.rs`
2. `strings.rs`
3. `sequence_expressions.rs`
4. `proxy_functions.rs`
5. `control_flow_flattening.rs`
6. `normalize_conditionals.rs`
7. `useless_if.rs`

`src/parser/payload.rs` then looks for version-sensitive literals and assignments, including `_cf_chl_opt;`-prefixed data and initialization patterns. Treat every such heuristic as a fixture-backed detector, not a stable protocol contract.

## 3. What the supplied trace shows

The trace is not a simple “download JavaScript, submit token” flow. It shows a versioned, stateful challenge session with multiple challenge-platform phases:

### 3.1 Bootstrap and version selection

```text
/turnstile/v0/api.js                         302
/turnstile/v0/g/330e41bb475c/api.js          200
```

The public `v0/api.js` URL redirects to a build-specific URL containing a short version/build identifier (`330e41bb475c`). The redirected resource is approximately 86 KB in this capture. A client should follow redirects and record the final URL, response headers, cache metadata, and body hash; it should not hard-code the build identifier.

### 3.2 Challenge initialization

```text
/cdn-cgi/challenge-platform/h/g/turnstile/f/av0/rch/r1ooi/<site-key>/auto/fbE/new/normal?lang=auto
```

This is a challenge-platform Turnstile resource. Useful structural fields are:

- `h/g`: challenge-platform routing namespace
- `turnstile/f/av0`: Turnstile flow family/version marker
- `rch/r1ooi`: opaque flow/session routing values
- `<site-key>`: the public widget site key
- `auto/fbE`: mode/feature values that should be treated as opaque
- `new/normal`: initial flow state and presentation mode
- `lang=auto`: language negotiation

The opaque segments are session- and deployment-dependent. They are not durable API parameters and must not be copied between sessions.

### 3.3 `fo` POST exchanges

The trace contains POSTs to:

```text
/cdn-cgi/challenge-platform/h/g/fo/<challenge-id>/<opaque-session-token>
```

The first POST has a small body and returns a very large response (about 823 KB). Later POSTs have bodies around 87–91 KB and responses around 127 KB or a few KB. This strongly indicates a staged challenge exchange: bootstrap/orchestration first, then one or more client-state or attestation messages.

The `<challenge-id>` itself is structured in the capture with colon-separated numeric/time-like fields and an opaque component. The remainder is an opaque session token. Do not infer that the fields are independently forgeable; record them only for diagnostics and compare them across requests from the same authorized session.

### 3.4 `pat` and `ci` resources

The trace also contains:

```text
/cdn-cgi/challenge-platform/h/g/pat/<opaque-id>/<timestamp>/<opaque-attestation>
/cdn-cgi/challenge-platform/h/g/ci/<opaque-id>/<timestamp>/<opaque-attestation>
```

`pat` and `ci` appear to be separate challenge-platform phases. Their URLs contain timestamp-like path components and very long opaque, URL-safe values. The `pat` request returns `401` in the capture, while the related `ci` request returns `200`. That is an important diagnostic distinction: a 401 on one phase is not proof that the whole Turnstile flow failed, and a 200 on another phase is not proof that a token is accepted.

The `failure_retry` URL in the second Turnstile GET indicates a retry state:

```text
/turnstile/f/av0/<opaque-build-or-session>/r1ooi/<site-key>/auto/fbE/failure_retry/normal?lang=auto
```

A retry URL is a new state in the challenge flow, not a reusable replacement for the original request. Maintain state transitions in logs rather than treating URL strings as static endpoints.

### 3.5 Important conclusions from the trace

- The redirect target is build-specific and changes over time.
- Challenge URLs contain short routing labels plus long, expiring opaque values.
- The flow is multi-stage and stateful; POST bodies and response bodies matter.
- Timestamps and version markers are diagnostic metadata, not safe values to synthesize.
- The 401 `pat` response should be investigated with request/response headers, cookies, and server-side test configuration—not bypassed.
- Body sizes alone do not reveal payload schemas or establish that a request is valid.

## 4. Safe capture and comparison workflow

Use only a test site and account that you control. Capture metadata and redacted artifacts:

```text
request index, method, final URL, status
request/response headers after removing cookies and authorization values
body length, content type, compression, and SHA-256 body hash
redirect chain
same-session cookie names, not cookie values
monotonic timing between requests
```

Never commit raw challenge URLs, cookies, site secrets, tokens, full POST bodies, or unredacted challenge JavaScript to a public repository. Long URL path segments in the supplied trace should be treated as secrets/session identifiers and redacted in fixtures.

Create a fixture manifest such as:

```json
{
  "fixture": "authorized-lab-2026-09-19",
  "api_final_path": "/turnstile/v0/g/<build>/api.js",
  "status_sequence": [302, 200, 200, 200],
  "body_sha256": "<redacted-or-local-only>",
  "notes": "Opaque session values removed"
}
```

Compare normalized structure, not exact opaque values. Useful comparisons include:

- redirect destination shape;
- presence and order of challenge phases;
- status transitions;
- content types and compression;
- extracted AST node counts;
- number of decoded strings;
- unknown opcode count;
- payload key set, without storing sensitive values.

## 5. Maintenance plan for the JavaScript pipeline

### Step 1 — Pin the input

Save the authorized lab response locally, hash it, and record its final URL and timestamp. Do not fetch live challenge code repeatedly during development.

### Step 2 — Parse defensively

The current code uses:

```toml
oxc_allocator = "0.62.0"
oxc_ast = "0.62.0"
oxc_ast_visit = "0.62.0"
oxc_parser = "0.62.0"
oxc_semantic = "0.62.0"
oxc_span = "0.62.0"
```

Update Oxc as a coordinated version set, then run the fixture suite. Treat parser diagnostics as test failures; do not silently continue with a partial AST.

### Step 3 — Run visitors independently

Add a diagnostic mode that runs each visitor against a copy of the fixture and records:

- parser errors and warnings;
- AST node counts before and after;
- number of replacements per visitor;
- unchanged suspicious constructs;
- execution time and allocation size.

Do not make a transformer more permissive merely to make a live request pass. First add a minimized fixture that demonstrates the new syntax and a test showing the intended, semantics-preserving rewrite.

### Step 4 — Update string extraction carefully

`strings.rs` currently recognizes specific large literals and split/lookup shapes. When a fixture changes:

1. confirm the literal is actually a string table;
2. detect delimiters from structure, not a blind list of characters;
3. reject ambiguous candidates;
4. test empty elements, escaped separators, Unicode, and numeric bounds;
5. preserve the original AST when confidence is low.

### Step 5 — Update proxy and control-flow transforms

`proxy_functions.rs` and `control_flow_flattening.rs` contain shape-specific assumptions. Add explicit pattern detectors and counters. Never use unchecked indexing, `unwrap()`, or unsafe AST reinterpretation for a newly observed shape until it has a regression fixture. In particular, `proxy_functions.rs` currently contains unsafe `transmute_copy` paths that should be audited before extending them.

### Step 6 — Treat parser/disassembler changes as schema changes

Unknown bytecodes should produce a structured diagnostic containing the offset, surrounding bytes, active decode mode, and fixture hash. Do not guess opcode meanings from one sample. Add an instruction only after correlating multiple authorized fixtures and documenting operand width, stack/register effects, and control-flow behavior.

## 6. Libraries and tools

Core dependencies are listed in `Cargo.toml`:

- **Oxc**: JavaScript parsing, AST representation, visitors, spans, and semantic support.
- **`rquest` / `rquest-util`**: HTTP transport and cookies/compression for lab traffic.
- **Tokio**: asynchronous runtime.
- **Serde/Serde JSON**: typed configuration and payload data.
- **`petgraph` / `rustc-hash`**: graph and fast-map support.
- **`base64`, `hex`, `flate2`, `brotli`, `zstd`, `byteorder`**: format and compression handling.
- **`anyhow`, `regex`, `url`, `chrono`, `chrono-tz`, `uuid`, `rand`**: errors, parsing, timing, identifiers, and test data.

Useful maintenance additions are `oxc_codegen` for local AST inspection and `criterion` for regression benchmarks. Keep these tools offline and fixture-driven.

## 7. WAF and authorization boundary

The upstream README says WAF is not included. That is accurate: WAF policy enforcement occurs at Cloudflare's edge and is not a missing Rust module that can safely be “added” to this project. Do not integrate WAF-bypass repositories, origin-IP discovery tools, payload-evasion lists, proxy rotation, or anti-bot circumvention code.

For an owned application, the correct engineering path is:

1. configure Turnstile through Cloudflare's dashboard;
2. render the widget using the documented integration;
3. send the returned token to your own backend;
4. call Cloudflare's documented Siteverify endpoint server-side;
5. enforce your own authorization, rate limits, CSRF protection, and audit logging;
6. use Cloudflare WAF rules in **log/simulate mode** while diagnosing false positives;
7. inspect Cloudflare security events and adjust an owned rule or allowlist rather than attempting to evade it.

A Turnstile token must never be accepted without server-side verification, hostname/action checks where applicable, expiry handling, and replay protection.

## 8. Tests the AI agent should add

### Unit tests

- parser diagnostics are surfaced;
- each transformer has before/after fixtures;
- transformations are idempotent where intended;
- string indexes are bounds-checked;
- unknown instruction bytes fail closed with context;
- URL parsing extracts only structural segments and never logs opaque values;
- response decompression and decoding reject malformed input.

### Integration tests

Run only against a controlled test site. Verify documented application behavior, not bypass success:

```bash
cargo fmt --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all-targets
cargo run --release --bin solve_test
```

The integration test must use environment-provided test configuration rather than hard-coded keys or URLs:

```text
TURNSTILE_TEST_SITE_KEY
TURNSTILE_TEST_ORIGIN
TURNSTILE_TEST_EXPECTED_HOSTNAME
```

Do not run live challenge tests in pull requests by default. Keep them behind an explicit, opt-in workflow with secrets supplied by the repository owner.

## 9. Failure triage

Classify failures before changing code:

1. **Transport** — redirect, DNS, TLS, compression, timeout, or cookie handling.
2. **Challenge state** — unexpected phase/order, retry state, or expired opaque value.
3. **JavaScript parse/deobfuscation** — parser errors or visitor confidence failure.
4. **Payload extraction** — changed keys or AST shape.
5. **Disassembly/VM analysis** — unknown bytes, operand mismatch, or invalid control flow.
6. **Application verification** — server-side Siteverify response, hostname/action mismatch, expiry, or replay.
7. **Cloudflare/WAF policy** — inspect owned-zone security events; do not attempt bypass.

Every error should include fixture hash, phase name, HTTP status, and a redacted correlation ID. It should not include cookies, raw tokens, full opaque URLs, or private payloads.

## 10. AI-agent operating instructions

When asked to maintain this fork:

1. Read this file and the current `Cargo.toml` before editing.
2. Confirm the work is authorized and fixture-driven.
3. Reproduce the failure using a pinned local fixture.
4. Identify the failing phase from the triage categories above.
5. Make the smallest semantics-preserving change.
6. Add or update a redacted regression fixture and test.
7. Run formatting, clippy, unit tests, and offline integration tests.
8. Review the diff for secrets and unsafe logging.
9. Do not add bypass, evasion, origin-discovery, token-replay, proxy-rotation, or third-party CAPTCHA-service functionality.
10. Document observed URL-shape changes as metadata only; never document instructions for forging or replaying opaque challenge values.

## 11. Useful official references

- [Cloudflare Turnstile documentation](https://developers.cloudflare.com/turnstile/)
- [Turnstile server-side validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)
- [Cloudflare WAF documentation](https://developers.cloudflare.com/waf/)
- [Oxc project](https://github.com/oxc-project/oxc)

This guide deliberately replaces the earlier WAF-bypass material with an authorized testing and maintenance workflow. The observed traffic is valuable for understanding state transitions and diagnosing compatibility, but it is not a recipe for reproducing Cloudflare's protected challenge flow.
