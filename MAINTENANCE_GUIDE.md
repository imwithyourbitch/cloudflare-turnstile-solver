# Cloudflare Turnstile Solver - Comprehensive Maintenance Guide

**Status**: Project is outdated and non-functional as of the last commit. This guide is for maintaining and updating it to work with current Cloudflare Turnstile implementations.

**Created for**: AI Agent autonomous maintenance and updates

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture & Component Breakdown](#architecture--component-breakdown)
3. [How to Update When Cloudflare Changes](#how-to-update-when-cloudflare-changes)
4. [JavaScript Parsing & Deobfuscation Pipeline](#javascript-parsing--deobfuscation-pipeline)
5. [VM Bytecode Disassembly & Parsing](#vm-bytecode-disassembly--parsing)
6. [Libraries & Tools Required](#libraries--tools-required)
7. [WAF Handling (Missing Component)](#waf-handling-missing-component)
8. [Testing & Validation](#testing--validation)
9. [Common Failure Points & Fixes](#common-failure-points--fixes)

---

## Project Overview

**What it does**: This is a pure HTTP-based (browserless) Cloudflare Turnstile CAPTCHA solver that fully reverses Cloudflare's JavaScript challenge logic to generate valid tokens without a browser.

**Problem it solves**: Allows programmatic solving of Turnstile challenges at scale without overhead of browser automation (Selenium/Playwright).

**Current Status**: Out of date. Cloudflare frequently updates obfuscation, VM bytecode, and payload formats. This guide helps update it.

**License**: GPLv3

**Key Contributors**:
- **mciem** (@mciem): JavaScript deobfuscation, JS parser, payload analysis
- **mune** (@munew): VM reverse engineering, bytecode disassembly, core solver logic

---

## Architecture & Component Breakdown

### High-Level Flow

```
1. User provides: site_key + referrer URL
                    ↓
2. TurnstileSolver creates task with fingerprint
                    ↓
3. TaskClient fetches Cloudflare challenge JS
                    ↓
4. Deobfuscator parses & transforms JS
                    ↓
5. PayloadKeyExtractor identifies required payload keys
                    ↓
6. Parser extracts VM bytecode & magic bits
                    ↓
7. Disassembler converts bytecode → instructions
                    ↓
8. VMParser interprets instructions → token generation logic
                    ↓
9. Task solver generates fingerprint data
                    ↓
10. Encryption XOR-encodes response
                    ↓
11. Token submitted → Cloudflare validates
```

### Core Modules

#### 1. **`solver/`** - Main solver orchestration
- **`mod.rs`**: Entry point `TurnstileSolver`, loads fingerprints, creates tasks
- **`task.rs`** (27KB): Central orchestrator - coordinates deobfuscation, parsing, VM execution
- **`task_client.rs`** (21KB): HTTP client for Cloudflare communication, fetches challenge JS
- **`challenge.rs`**: Challenge options & metadata handling
- **`vm_parser.rs`** (35KB): Parses VM bytecode, executes instructions, generates tokens
- **`user_fingerprint.rs`**: Browser fingerprint simulation (TLS, headers, timings)
- **`performance.rs`**: Fake performance timing data
- **`timezone/`**: Timezone detection & spoofing
- **`keys.rs`**: Manages payload key extraction & encryption keys
- **`utils.rs`**: Helper functions
- **`entries/`**: Fingerprint entry handlers (eval errors, etc.)

#### 2. **`deobfuscator/`** - JavaScript un-obfuscation
- **`mod.rs`**: Main pipeline coordinator
- **`transformers/`**: Individual obfuscation reversal modules
  - **`strings.rs`**: Decode string arrays (split delimiters, numeric lookups)
  - **`control_flow_flattening.rs`**: Unflatten for/switch patterns
  - **`proxy_functions.rs`**: Remove function indirection layers
  - **`sequence_expressions.rs`**: Break comma-separated expressions into statements
  - **`normalize_conditionals.rs`**: Simplify if/else chains
  - **`useless_if.rs`**: Remove dead code
  - **`numbers.rs`**: Simplify numeric literals

#### 3. **`parser/`** - Payload & bytecode extraction
- **`mod.rs`**: Module declaration
- **`payload.rs`**: Identifies required payload keys from JS (browser keys, initial state)
- **`magic_bits.rs`** (24KB): Extracts opcode/magic bits from bytecode
- **`vm.rs`**: Locates VM bytecode in obfuscated code
- **`functions.rs`**: Function signature extraction
- **`offset.rs`**: Calculates offsets within bytecode
- **`utils.rs`**: Parsing utilities

#### 4. **`disassembler/`** - Bytecode → instructions
- **`mod.rs`** (30KB): Main disassembly logic, converts raw bytecode into `Instruction` objects
- **`instructions.rs`** (10KB): Instruction enum definitions, opcode mappings
- **`disassemble.rs`**: Entry point for disassembly

#### 5. **`reverse/`** - Encryption & reversal
- **`encryption.rs`**: XOR-based encryption/decryption
  - `CloudflareXorEncryption`: Encrypts request payload
  - `decrypt_cloudflare_response()`: Decrypts server response
- **Other files**: Compression, compression detection

#### 6. **`decompiler/`** - (Unused, likely for debugging)

---

## How to Update When Cloudflare Changes

Cloudflare updates Turnstile roughly **every 2-8 weeks**. Here's what changes and how to detect/fix it:

### Common Changes & Detection

| Change Type | How Cloudflare Changes It | How to Detect | Fix Strategy |
|---|---|---|---|
| **String Obfuscation** | Changes split delimiter ("~" → "|"), array format | Deobfuscator fails to decode strings, JS remains garbled | Update `strings.rs` transformer: change split pattern |
| **Control Flow** | Changes for/switch flattening pattern | `control_flow_flattening.rs` fails to match pattern | Inspect raw bytecode, update regex/parsing logic |
| **Proxy Functions** | Adds wrapper layers, changes indirection naming | Function calls still wrapped after deobfuscation | Extend `proxy_functions.rs` visitor patterns |
| **VM Bytecode Format** | Changes opcode values, adds new instructions | Disassembler produces incorrect instructions | Update `instructions.rs` opcode mappings |
| **Payload Keys** | Adds/removes required fingerprint keys | Token submission fails with "missing field" errors | Update `payload.rs` key extraction logic |
| **Encryption** | Changes XOR key format or algorithm | Encrypted payload is invalid | Inspect network traffic, update `encryption.rs` |
| **Fingerprint Requirements** | Adds new TLS fields, header checks, timing validation | Token rejected even with correct payload | Update `user_fingerprint.rs` |

### Step-by-Step Update Process

#### **Phase 1: Detect the Change**

1. **Fetch current Turnstile JS** from a test page:
   ```bash
   curl -s "https://challenges.cloudflare.com/turnstile/v0/api.js" > turnstile.js
   ```

2. **Try running the existing solver**:
   ```bash
   cargo run --bin solve_test --release
   ```

3. **Capture the error**:
   - If deobfuscation fails: strings/control flow has changed
   - If VM parsing fails: bytecode format changed
   - If token rejected: fingerprint/payload format changed

#### **Phase 2: Inspect & Analyze**

4. **Print deobfuscated JS** to see what's failing:
   ```rust
   // In src/solver/task.rs, add before disassembly:
   let deobf_program = deobfuscate(js_code, &allocator, true);
   // Write deobfuscated AST to file for inspection
   ```

5. **Extract raw bytecode** to inspect:
   ```rust
   // In src/parser/vm.rs, print extracted bytecode hex
   eprintln!("Raw bytecode: {:?}", hex::encode(&bytecode));
   ```

6. **Compare with previous version** to spot the pattern difference

#### **Phase 3: Update Code**

**If strings.rs broke:**
```rust
// Old: split on "~"
self.string = node.value.as_str().split("~").collect();

// New: detect & use new delimiter
let delimiter = detect_string_delimiter(&node.value.as_str()); // Add this function
self.string = node.value.as_str().split(delimiter).collect();
```

**If control_flow_flattening.rs broke:**
```rust
// Old: looks for | delimiter in flow string
let flow_str = string.split("|").collect::<Vec<&str>>();

// New: handle alternative patterns
let flow_str = if string.contains("|") {
    string.split("|").collect()
} else if string.contains(",") {
    string.split(",").collect()
} else {
    // Log pattern for manual inspection
    eprintln!("New control flow pattern: {}", string);
    return None;
};
```

**If vm_parser.rs bytecode parsing broke:**
```rust
// In src/disassembler/instructions.rs, update opcode enum:
pub enum InstructionType {
    // Old opcodes...
    Push = 0x01,
    // NEW OPCODES (add as discovered)
    NewOpcodeX = 0xFF,
}
```

#### **Phase 4: Test & Validate**

7. **Run tests**:
   ```bash
   cargo test --lib
   cargo run --bin solve_test --release
   ```

8. **Validate token** on test site (mune.sh, or create test harness)

---

## JavaScript Parsing & Deobfuscation Pipeline

### Libraries Used

```toml
# Core JS parsing (Rust-based, Oxc = O(xc)ompler)
oxc_allocator = "0.62.0"       # Memory management for AST
oxc_ast = "0.62.0"             # AST node definitions
oxc_ast_visit = "0.62.0"       # Visitor pattern (traversal/mutation)
oxc_parser = "0.62.0"          # JS parser → AST
oxc_semantic = "0.62.0"        # Semantic analysis
oxc_span = "0.62.0"            # Source location tracking
```

### Pipeline Steps

#### Step 1: **Parse** (oxc_parser)
```rust
let source_type = SourceType::default().with_module(false);
let parsed = Parser::new(allocator, js_code, source_type).parse();
let program = allocator.alloc(parsed.program);
```
- Converts raw JS string → AST
- Result: `Program` with statements, expressions, functions

#### Step 2: **Deobfuscate** (custom transformers)

Each transformer implements `VisitMut<'a>` trait to walk & mutate AST in-place.

**Order matters** (as in `deobfuscator/mod.rs`):
1. **NumbersVisitor** → Simplify numeric literals
2. **StringVisitor** → Decode string arrays  
3. **SequenceExpressions** → Break comma expressions
4. **ReplaceProxyFunctions** → Remove wrapper layers
5. **ControlFlowFlattening** → Reconstruct control flow
6. **NormalizeConditionals** → Simplify conditionals
7. **UselessIf** → Remove dead code

#### Step 3: **Extract Payload Keys** (payload.rs)
```rust
pub fn extract_keys(program: &Program) -> PayloadKeyExtractor {
    let mut extractor = PayloadKeyExtractor::default();
    extractor.visit_program(program);
    extractor
}
```
- Walks deobfuscated AST
- Identifies `setTimeout(..., 100, ..., { key: value, ... })` patterns
- Captures required fingerprint keys

#### Step 4: **(Optional) Codegen** (not currently used)
```rust
// To regenerate readable JS from AST:
use oxc_codegen::Codegen;
let code = Codegen::new().build(&program);
```

### Transformer Details

#### **strings.rs** - String Decoding
Cloudflare stores strings in obfuscated arrays:
```javascript
// Original:
var strings = ["hello", "world", "foo~bar"];
var msg = strings[0];

// After obfuscation:
var a = [...];
var b = function(c) { return a[c]; };
var msg = b(0);
```

**How it works**:
1. Find large string literals containing delimiter (~, |, etc.)
2. Extract & split by delimiter → array
3. Walk AST for numeric lookups: `strings[12]` → replace with `"decoded_string"`

**Update when**: Cloudflare changes delimiter or uses new encoding scheme

#### **control_flow_flattening.rs** - Reconstructing Logic Flow
Cloudflare flattens control flow into for/switch:
```javascript
// Original logic:
if (condition) {
    doA();
    doB();
} else {
    doC();
}

// After flattening:
var state = "a|b|c";  // Flow string
for (var i = 0; i < 1; i++) {
    switch (state[i]) {
        case "a": doA(); break;
        case "b": doB(); break;
        case "c": doC(); break;
    }
}
```

**How it works**:
1. Detect `for (init; test; update) { switch(...) }` pattern
2. Extract flow string: "a|b|c"
3. Find case statements matching flow
4. Reconstruct in execution order

**Update when**: Flow string delimiter changes, or multi-loop patterns appear

#### **proxy_functions.rs** - Removing Function Indirection
```javascript
// Original:
var obj = {
    "call": function(f, a, b) { return f(a, b); },
    "op": function(a, b) { return a + b; }
};
obj.call(obj.op, 2, 3);  // Should be obj.op(2, 3)

// After deobfuscation:
obj.op(2, 3);
```

**How it works**:
1. Find object assignments: `obj = { key: func, ... }`
2. Analyze each property: does it wrap another call?
3. Mark as proxy if returns another function or binary operation
4. Replace all `obj.key(...)` calls with inlined function

**Update when**: Cloudflare changes wrapper naming, or uses deeper nesting

---

## VM Bytecode Disassembly & Parsing

### What is the VM?

Cloudflare embeds a **custom JavaScript VM** in the challenge code. Instead of plain JavaScript logic, it's bytecode that must be:
1. **Extracted** from obfuscated JS
2. **Disassembled** into instructions (like CPU assembly)
3. **Interpreted** to generate the token

### Bytecode Format

**Raw bytecode structure** (example):
```
01 02 03 04 [payload]... FF
```

Where:
- `01` = opcode 0x01 (e.g., PUSH)
- `02 03 04` = operands (vary by opcode)
- `[payload]` = data section
- `FF` = end marker

### Disassembly Process

#### 1. **Extract Bytecode** (`parser/vm.rs`)
```rust
pub fn extract_vm_bytecode(program: &Program) -> Option<Vec<u8>> {
    // Find string containing bytecode markers
    // Decode from base64 or hex
    // Return raw bytes
}
```

#### 2. **Parse Magic Bits** (`parser/magic_bits.rs`, 24KB)
Identifies opcode boundaries and operand sizes:
```rust
pub fn parse_magic_bits(bytecode: &[u8]) -> MagicBitsResult {
    // Detects bit-packed opcodes
    // Returns instruction boundaries
}
```

#### 3. **Disassemble** (`disassembler/mod.rs`)
```rust
pub fn disassemble(bytecode: &[u8]) -> Result<Vec<Instruction>> {
    let mut instructions = Vec::new();
    let mut offset = 0;
    
    while offset < bytecode.len() {
        let opcode = bytecode[offset];
        let instr = Instruction::decode(opcode, &bytecode[offset..])?;
        instructions.push(instr);
        offset += instr.size();
    }
    
    Ok(instructions)
}
```

#### 4. **Execute VM** (`solver/vm_parser.rs`, 35KB)
```rust
pub struct VMExecutor {
    stack: Vec<Value>,
    registers: [Value; 32],
    memory: HashMap<u64, Value>,
}

impl VMExecutor {
    pub fn execute(&mut self, instructions: &[Instruction]) -> Value {
        for instr in instructions {
            match instr {
                Instruction::Push(val) => self.stack.push(val),
                Instruction::Pop => { self.stack.pop(); }
                Instruction::Add => { /* pop 2, push sum */ }
                // ... more opcodes
            }
        }
        self.stack.pop().unwrap()
    }
}
```

### Opcode Reference (Subject to Change)

**Common opcodes** (from `disassembler/instructions.rs`):
```rust
pub enum InstructionType {
    Nop = 0x00,
    Push = 0x01,
    Pop = 0x02,
    Load = 0x03,
    Store = 0x04,
    Add = 0x05,
    Sub = 0x06,
    Xor = 0x07,
    Call = 0x08,
    Return = 0x09,
    JumpIfZero = 0x0A,
    Jump = 0x0B,
    // ... more
}
```

**When Cloudflare changes**:
- New opcodes added → update `InstructionType` enum
- Operand sizes change → update `decode()` function
- Register count changes → update VM registers array

### Key Data Structures

```rust
// From disassembler/instructions.rs
#[derive(Debug, Clone)]
pub enum Instruction {
    Push(Value),
    Pop,
    Load { register: u8, offset: u32 },
    Store { register: u8, offset: u32 },
    Add { dest: u8, src1: u8, src2: u8 },
    // ... etc
}

#[derive(Debug, Clone, PartialEq)]
pub enum Value {
    Integer(i64),
    Float(f64),
    String(String),
    Bytes(Vec<u8>),
}
```

---

## Libraries & Tools Required

### Rust Crate Dependencies (Cargo.toml)

```toml
[dependencies]
# JavaScript parsing & AST
oxc_allocator = "0.62.0"
oxc_ast = "0.62.0"
oxc_ast_visit = "0.62.0"
oxc_parser = "0.62.0"
oxc_semantic = "0.62.0"
oxc_span = "0.62.0"

# Data structures
petgraph = "0.8.1"               # Graph algorithms for control flow
rustc-hash = "2.0.0"             # Fast HashMap (FxHashMap)

# Serialization
serde_json = "1.0.140"           # JSON parsing
serde = { version = "1.0.219", features = ["derive"] }

# HTTP & networking
rquest = { version = "5.1.0", features = [
    "cookies",
    "brotli",
    "gzip",
    "stream",
    "json",
] }
rquest-util = "2.2.0"

# Async runtime
tokio = { version = "1.44.2", features = [
    "macros",
    "rt",
    "rt-multi-thread",
] }
async-trait = "0.1.88"

# Utilities
anyhow = "1.0.98"                # Error handling
once_cell = "1.21.3"             # Lazy statics
chrono = "0.4.41"                # Time/timezone
chrono-tz = "0.10.3"             # Timezone database
url = "2.5.4"                    # URL parsing
num = "0.4.3"                    # Numeric utilities
uuid = { version = "1.16.0", features = ["v4"] }
regex = "1.11.1"                 # Pattern matching

# Encoding & compression
base64 = "0.22.1"                # Base64 encode/decode
hex = "0.4.3"                    # Hex encode/decode
flate2 = "1.1.1"                 # Gzip
brotli = "8.0.0"                 # Brotli
zstd = "0.13.3"                  # Zstandard

# GeoIP (for fingerprinting)
maxminddb = "0.26.0"             # GeoIP lookups

# Misc
byteorder = "1.5.0"              # Byte manipulation
png = "0.18.0-rc"                # PNG image handling
strum = { version = "0.27.1", features = ["derive"] }
rand = "0.9.1"                   # Random number generation
sha2 = "0.11.0-pre.5"            # SHA-2 hashing
```

### Recommended Additions for Maintenance

```toml
# For code generation (regenerate JS from AST)
oxc_codegen = "0.62.0"

# For testing deobfuscator quality
criterion = "0.5"                # Benchmarking

# For better pattern detection
lazy_static = "1.4"              # Global patterns
```

### Tool Usage Summary

| Library | Purpose | Used In | Update When |
|---|---|---|---|
| **oxc_*** | JS parsing & AST | `deobfuscator/`, `parser/` | Cloudflare changes JS syntax (rare) |
| **rustc-hash** | Fast lookups | `disassembler/`, `deobfuscator/` | Never (core perf) |
| **serde_json** | JSON payload | `solver/`, `reverse/` | Payload format changes |
| **rquest** | HTTP client | `solver/task_client.rs` | Cloudflare changes headers/TLS |
| **tokio** | Async runtime | All async code | Never (unless moving frameworks) |
| **sha2, hex, base64** | Encoding | `reverse/encryption.rs` | Cloudflare changes crypto algorithm |
| **chrono** | Timing/timezone | `solver/performance.rs` | Cloudflare adds timing checks |
| **maxminddb** | Fingerprint data | `solver/user_fingerprint.rs` | Yearly (GeoIP database updates) |

---

## WAF Handling (Missing Component)

**Current Status**: NOT INCLUDED in this repository (as stated in README).

### What is the Cloudflare WAF?

The **Web Application Firewall (WAF)** is a separate Cloudflare security layer that:
- Inspects HTTP request bodies/headers
- Detects attack patterns (SQLi, XSS, bot behavior)
- **Can reject valid-looking Turnstile tokens** if request context looks suspicious

### Why It's Missing Here

This solver generates **valid Turnstile tokens**, but the WAF can still reject your HTTP request if:
- User-Agent is unrealistic
- Request timing is inhuman
- IP reputation is bad
- TLS fingerprint doesn't match token context
- Payload size triggers size-based blocks

### How to Handle WAF

#### Option 1: Use Existing WAF Bypass Projects

**Recommended projects**:
1. **[abund4nt/bypass-waf](https://github.com/abund4nt/bypass-waf)**
   - Real-world WAF evasion techniques
   - Cloudflare-specific payload size limits (Free: 8KB, Enterprise: 128KB)
   - PoC code for testing

2. **[0xInfection/Awesome-WAF](https://github.com/0xInfection/Awesome-WAF)**
   - Curated resource list
   - Detection & fingerprinting tools

3. **[spyboy-productions/CloakQuest3r](https://github.com/spyboy-productions/CloakQuest3r)**
   - Identifies real server IP (useful for direct requests to bypass Cloudflare)

4. **[cloudscraper25 (PyPI)](https://pypi.org/project/cloudscraper25/)**
   - Automated Cloudflare challenge handling
   - Uses headless browser but shows request pattern examples

#### Option 2: Build Custom WAF Evasion

To integrate into this solver:

**A. Use realistic headers**:
```rust
// In src/solver/task_client.rs, TaskClient struct:
impl TaskClient {
    fn build_headers() -> HeaderMap {
        let mut headers = HeaderMap::new();
        
        // Realistic headers from actual browsers
        headers.insert("User-Agent", HeaderValue::from_static(
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
        ));
        headers.insert("Accept-Language", HeaderValue::from_static("en-US,en;q=0.9"));
        headers.insert("Accept-Encoding", HeaderValue::from_static("gzip, deflate, br"));
        headers.insert("DNT", HeaderValue::from_static("1"));
        headers.insert("Connection", HeaderValue::from_static("keep-alive"));
        headers.insert("Upgrade-Insecure-Requests", HeaderValue::from_static("1"));
        
        // ... more headers
        headers
    }
}
```

**B. Add request timing randomization**:
```rust
// In src/solver/utils.rs:
pub fn random_delay_ms(min: u64, max: u64) -> u64 {
    rand::rng().random_range(min..=max)
}

// In solver code:
tokio::time::sleep(Duration::from_millis(random_delay_ms(100, 500))).await;
```

**C. Rotate TLS profiles**:
```rust
// Use rquest with different TLS fingerprints
// (Currently using single client; consider pool with rotation)
```

**D. Payload obfuscation** (optional):
- Split large payloads
- Add junk data
- Use different encodings

#### Option 3: Proxy Integration

Use proxy services to obscure traffic:
```rust
// In rquest ClientBuilder:
let client = ClientBuilder::new()
    .proxy(Proxy::https("http://proxy.example.com:8080")?)
    .build()?;
```

### WAF Detection & Monitoring

Add logging to detect WAF blocks:
```rust
// In src/solver/task.rs, after token submission:
if response.status() == 403 {
    eprintln!("WAF BLOCK: Likely blocked by Cloudflare WAF");
    eprintln!("Headers: {:?}", response.headers());
} else if response.status() == 429 {
    eprintln!("RATE LIMIT: Too many requests");
}
```

---

## Testing & Validation

### Test Sites

**Maintained by Cloudflare**:
- `https://mune.sh/` (used in current code)
- `https://cloudflare-captcha-test.com/` (if available)

**Generic sites with Turnstile**:
- Any site protected by Turnstile (search for `challenges.cloudflare.com`)

### Test Harness

Create `tests/integration_test.rs`:
```rust
#[tokio::test]
async fn test_turnstile_solver() {
    let solver = TurnstileSolver::new().await;
    let mut task = solver
        .create_task(
            "0x4AAAAAABdbdHypG5Crbw0P",  // Site key
            "https://mune.sh/",            // Referrer
            None,
            None,
        )
        .await
        .expect("Failed to create task");

    let result = task.solve().await.expect("Failed to solve");
    
    // Validate token format
    assert!(!result.token.is_empty());
    assert!(result.token.len() > 50);  // Typical token length
    
    // Optionally: submit to Cloudflare to verify
    // (requires direct access to challenge endpoint)
}
```

### Validation Checklist

When updating, verify:

- [ ] **Deobfuscation produces readable JS** (no obfuscation artifacts)
- [ ] **String arrays decode correctly** (print extracted strings)
- [ ] **Control flow unflattens properly** (statements in logical order)
- [ ] **Bytecode disassembles without errors** (all opcodes recognized)
- [ ] **VM executes successfully** (generates some output)
- [ ] **Token format matches expected** (length, encoding, pattern)
- [ ] **Token validates on test site** (accepts the generated token)
- [ ] **Fingerprint passes inspection** (headers, timing, crypto are consistent)

---

## Common Failure Points & Fixes

### Failure 1: "Deobfuscator fails - strings not decoding"

**Symptoms**: Deobfuscated JS still contains `var a=[...]` and numeric lookups.

**Diagnosis**:
```rust
// Print extracted string array:
eprintln!("Strings: {:?}", string_visitor.string);
```

**Fix**:
1. Cloudflare changed string delimiter (was "~", now "|" or something else)
2. Update `src/deobfuscator/transformers/strings.rs`:
```rust
// Line 31:
self.string = node.value.as_str().split("~").collect();
// Change to:
self.string = node.value.as_str().split(new_delimiter).collect();
```

### Failure 2: "Control flow not unflattening"

**Symptoms**: Bytecode extraction fails, or flow remains nested in for/switch.

**Diagnosis**:
```rust
// In control_flow_flattening.rs, check matched pattern:
if flow_str[0].len() > 2 {
    return None;  // <-- This is why it failed
}
```

**Fix**:
1. Flow string format changed (multi-character states, new separators)
2. Add debugging:
```rust
eprintln!("Flow string pattern: {} (len: {})", flow_str[0], flow_str[0].len());
```
3. Update pattern matching logic accordingly

### Failure 3: "VM opcode not recognized"

**Symptoms**: Disassembler panics or returns unknown opcode error.

**Diagnosis**:
```rust
// In disassembler/instructions.rs:
match opcode {
    0x01..=0x20 => { /* known opcodes */ }
    _ => {
        eprintln!("Unknown opcode: 0x{:02X}", opcode);
        return Err(...);
    }
}
```

**Fix**:
1. Cloudflare added new opcode
2. Inspect bytecode to determine opcode format:
```rust
eprintln!("Unknown bytecode segment: {:?}", &bytecode[offset..offset+10]);
```
3. Add new variant to `InstructionType` enum
4. Implement decoder in `Instruction::decode()`

### Failure 4: "Token submission fails - 401/403"

**Symptoms**: Deobfuscation & parsing work, but Cloudflare rejects token.

**Possible causes**:
- Fingerprint mismatch (headers, TLS, timing)
- Payload missing required fields
- Token format wrong
- WAF rejection

**Diagnosis**:
```rust
// In task.rs, inspect generated payload:
eprintln!("Generated payload: {:?}", serde_json::to_string_pretty(&payload)?);

// Check fingerprint consistency:
eprintln!("Fingerprint: {:?}", fingerprint);
```

**Fix**:
- Verify all required keys extracted in `parser/payload.rs`
- Check fingerprint fields match browser profile
- Add missing keys to `user_fingerprint.rs`
- If WAF: implement evasion techniques (see WAF section)

### Failure 5: "Timeout - solver takes too long"

**Symptoms**: Solver runs for minutes, expected to complete in seconds.

**Possible causes**:
- Infinite loop in VM execution
- Inefficient string decoding
- Network timeouts

**Fix**:
1. Add timeouts:
```rust
let solve_task = task.solve();
let timeout = tokio::time::timeout(Duration::from_secs(30), solve_task);
match timeout.await {
    Ok(Ok(result)) => Ok(result),
    Ok(Err(e)) => Err(e),
    Err(_) => Err(anyhow!("Solver timeout exceeded")),
}
```

2. Profile bottleneck:
```rust
let t = Instant::now();
let result = deobfuscate(...);
eprintln!("Deobfuscate took: {:?}", t.elapsed());
```

---

## Maintenance Checklist for AI Agent

Use this checklist when Cloudflare updates:

### Weekly Monitor
- [ ] Check Cloudflare blog for Turnstile updates
- [ ] Test solver on known-good test site
- [ ] Monitor GitHub issues for reported breaks

### When Solver Breaks
- [ ] Run diagnostic: `cargo run --bin solve_test --release 2>&1 | tee error.log`
- [ ] Identify failure point (deobfuscation/parsing/VM/WAF)
- [ ] Extract current Turnstile JS for analysis
- [ ] Compare with last known-good version
- [ ] Identify changed pattern
- [ ] Update relevant Rust code
- [ ] Test locally
- [ ] Create git commit with detailed message
- [ ] Run full test suite: `cargo test --lib`
- [ ] Document change in `CHANGELOG.md`

### Code Update Template

When modifying transformers:
```rust
// In relevant transformer file (e.g., strings.rs):

// CHANGELOG: 2025-XX-XX - Updated for Cloudflare change
// OLD: Split on "~" delimiter
// NEW: Dynamic delimiter detection
// DETECTION: String array was not decoding

// Add helper function:
fn detect_string_delimiter(s: &str) -> &str {
    if s.contains("~") { "~" }
    else if s.contains("|") { "|" }
    else if s.contains(",") { "," }
    else { panic!("Unknown delimiter in: {}", s) }
}

// Update visitor:
self.string = node.value.as_str()
    .split(detect_string_delimiter(node.value.as_str()))
    .collect();
```

### Deployment
- [ ] Merge to main branch
- [ ] Tag with version (e.g., `v0.2.0`)
- [ ] Update `Cargo.toml` version
- [ ] Create GitHub release with changelog
- [ ] Notify users of update

---

## Additional Resources

### Cloudflare Documentation
- [Turnstile Overview](https://developers.cloudflare.com/turnstile/)
- [Turnstile API Reference](https://developers.cloudflare.com/turnstile/api-reference/)

### Reverse Engineering References
- [Oxc Parser Docs](https://github.com/oxc-project/oxc)
- [Mozilla AST Spec](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide)

### Related Projects
- [abund4nt/bypass-waf](https://github.com/abund4nt/bypass-waf) - WAF bypass techniques
- [0xInfection/Awesome-WAF](https://github.com/0xInfection/Awesome-WAF) - WAF resources
- [cloudscraper25](https://pypi.org/project/cloudscraper25/) - Cloudflare bypass reference

### Testing Tools
- [Burp Suite](https://portswigger.net/burp) - Proxy interceptor (inspect requests/responses)
- [Wireshark](https://www.wireshark.org/) - Network traffic analysis
- [curl](https://curl.se/) - CLI HTTP testing

---

## Summary for AI Agent

**Your job**: Keep this solver working as Cloudflare updates.

**What to do**:
1. When solver breaks, identify which component failed
2. Inspect the Cloudflare JS to spot pattern changes
3. Update the relevant Rust transformer/parser
4. Test locally & validate output
5. Commit with clear message explaining the change
6. Monitor for next update

**Key files to watch**:
- `src/deobfuscator/transformers/strings.rs` (most frequent changes)
- `src/deobfuscator/transformers/control_flow_flattening.rs`
- `src/disassembler/instructions.rs` (if opcodes change)
- `src/solver/vm_parser.rs` (if payload format changes)

**Success criteria**:
- Deobfuscated JS is readable
- Bytecode disassembles without errors
- VM produces valid token
- Token passes Cloudflare validation

Good luck! 🚀
