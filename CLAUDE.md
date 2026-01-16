# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Knus is a Rust parser library for the [KDL](https://kdl.dev) (KDL Document Language) file format. It provides:
- High-quality error reporting via `miette`
- Derive macros (`#[derive(knus::Decode)]`) for automatic deserialization
- Almost full KDL v2 specification compliance

## Common Commands

```bash
# Run all tests
cargo test --workspace

# Lint (format check + clippy)
cargo fmt --check && cargo clippy --workspace --tests

# Watch mode (auto-test on changes)
just watch

# Generate coverage report
just cov

# Check WASM compatibility
cargo check --workspace --target wasm32-unknown-unknown

# Test with minimal dependency versions
cargo +nightly test --workspace -Z direct-minimal-versions
```

Run a single test:
```bash
cargo test --workspace <test_name>
```

## Architecture

The library uses a **two-phase parsing pattern**:

```
KDL Text → [Chumsky Parser] → AST → [Decode Traits] → User's Rust Types
```

### Workspace Structure

- **Root crate (`knus`)** - Parser and AST implementation
- **`derive/` crate (`knus-derive`)** - Procedural macros

### Key Modules (in `src/`)

| Module | Purpose |
|--------|---------|
| `grammar.rs` | Chumsky parser combinators implementing KDL v2 spec |
| `ast.rs` | AST types: `Node`, `Value`, `Literal`, `Spanned<T>` |
| `traits.rs` | Core traits: `Decode`, `DecodeScalar`, `DecodeChildren`, `DecodePartial` |
| `errors.rs` | Error types with `miette::Diagnostic` integration |
| `decode.rs` | `Context` for error accumulation and decode helpers |
| `span.rs` | Byte offset tracking, lazy line/column computation |
| `wrappers.rs` | Public API: `parse()`, `parse_ast()`, `parse_with_context()` |
| `convert.rs` | Numeric type conversions for all integer/float types |

### Derive Macro System (in `derive/src/`)

| Module | Purpose |
|--------|---------|
| `definition.rs` | Parse `#[knus(...)]` attributes into structured form |
| `node.rs` | Generate `Decode` impl for structs/enums |
| `scalar.rs` | Generate `DecodeScalar` impl |
| `variants.rs` | Handle enum variant matching |

### Core Derive Attributes

```rust
#[knus(argument)]              // Positional argument
#[knus(arguments)]             // Remaining arguments as Vec
#[knus(property)]              // Named property (uses field name)
#[knus(property = "name")]     // Named property with explicit name
#[knus(child)]                 // Single child node
#[knus(children)]              // Multiple children as Vec
#[knus(children(name = "x"))]  // Children filtered by node name
#[knus(node_name)]             // Extract node name
#[knus(type_name)]             // Extract type annotation
#[knus(flatten)]               // Merge nested structures
#[knus(unwrap(argument))]      // Extract from nested structure
#[knus(default)]               // Use Default::default() if missing
```

## Cargo Features

- `derive` (default) - Procedural macro support
- `base64` (default) - Base64 encoding/decoding
- `line-numbers` (default) - Line/column calculation via `unicode-width`
- `minicbor` - CBOR serialization support

## Key Patterns

**Error Accumulation**: `Context` collects multiple errors instead of failing fast, allowing users to see all problems at once.

**Type Annotations**: KDL's `(typename)value` syntax is validated during decode (non-fatal errors).
