---
name: rust-development
description: Use the Rust compiler (rustc, cargo) to compile and test code in a way that reduces AI tokens consumption.
---

# Rust Compiler Skill

This skill optimizes Rust compilation workflows to minimize token consumption while maintaining full visibility into errors and warnings.

## Core Principle
For most development tasks use the rust lsp (rust-analyzer) for real-time feedback. When you need to compile or run tests, use the following patterns to capture only relevant output while discarding verbose messages.
Rust compiler output can be extremely verbose, especially for successful builds or when downloading dependencies. Redirect stdout to `/dev/null` while preserving stderr for actual errors and warnings.

## Essential Commands

### Compilation Check (Fastest Feedback)

```bash
cargo check 2>&1 >/dev/null | head -100
```

Use `cargo check` instead of `cargo build` when you only need to verify code compiles. It skips code generation and is significantly faster.

### Building Code

```bash
cargo build 2>&1 >/dev/null | head -100
```

For release builds:

```bash
cargo build --release 2>&1 >/dev/null | head -100
```

### Clippy Linting

```bash
cargo clippy 2>&1 >/dev/null | head -100
```

Always run clippy after making changes. All warnings must be addressed.

### Running Tests

```bash
cargo test 2>&1 >/dev/null | head -150
```

For a specific test:

```bash
cargo test test_name 2>&1 >/dev/null | head -100
```

For a specific crate in a workspace:

```bash
cargo test -p crate-name 2>&1 >/dev/null | head -100
```

### Formatting

```bash
cargo fmt --all
```

Always run after editing Rust files. No output redirection needed as it produces minimal output.

## Workspace-Specific Commands

For large workspaces, target specific crates to reduce compilation scope:

```bash
# Check a specific crate
cargo check -p crate-name 2>&1 >/dev/null | head -100

# Build a specific binary
cargo build --bin binary-name 2>&1 >/dev/null | head -100

# Run clippy on a specific crate
cargo clippy -p crate-name 2>&1 >/dev/null | head -100
```

## Error Investigation

When errors occur, you may need more context. Use these patterns:

```bash
# Get more error lines if truncated
cargo check 2>&1 >/dev/null | head -200

# See full error for a specific issue
cargo check 2>&1 >/dev/null | grep -A 20 "error\[E"

# Count total errors
cargo check 2>&1 >/dev/null | grep -c "^error"
```

## Recommended Workflow

1. **After writing/editing code**: Run `cargo fmt --all`
2. **Quick validation**: Run `cargo check -p <crate> 2>&1 >/dev/null | head -100`
3. **Before committing**: Run `cargo clippy 2>&1 >/dev/null | head -100`
4. **For tests**: Run `cargo test -p <crate> 2>&1 >/dev/null | head -150`

## Why Truncate Output?

- Successful cargo builds can produce thousands of lines of "Compiling..." messages
- Downloading dependencies generates extensive output
- Using `head -100` or `head -150` captures errors while avoiding token bloat
- The pattern `2>&1 >/dev/null` redirects stderr to the pipe while discarding stdout, ensuring only errors and warnings are captured
