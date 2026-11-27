# Rust Style Guide

This guide documents cross-project Rust conventions for module organization, imports, visibility, type design, error semantics, public API documentation, and testing. Standard formatting and baseline lints are delegated to `rustfmt` and `clippy`; the rules below cover deliberate choices where valid Rust styles diverge.

## Module Layout and Ordering

- **File header order**: Order the top of each module file as module documentation (`//!`), ordinary `use` imports, out-of-line `mod` declarations (with any immediate `pub use self::...` re-exports), and module items. Placing `use` blocks before `mod` declarations ensures IDE and agent auto-imports append cleanly to existing import groups.
- **Directory modules**: Organize multi-file modules under `<module>/mod.rs` rather than mixing `<module>.rs` with a sibling `<module>/` directory.
- **File splitting restraint**: Keep cohesive types and helpers in one file until a distinct responsibility emerges; a separate file does not by itself justify widening visibility boundaries. Extract a private companion file (such as `impls.rs` for repetitive trait forwarding or tuple macro implementations) only when isolating boilerplate clarifies the primary module.
- **Grouping `mod` declarations**: Group out-of-line `mod` declarations by role, separated by a blank line and sorted alphabetically within each group:
  1. Public modules (`pub mod ...;`).
  2. Private modules with public re-exports (`mod foo;` followed immediately by `pub use self::foo::...;`).
  3. Implementation-only private modules (`mod ...;`), with out-of-line `#[cfg(test)]` modules last.
- **Subsystem facades**: When a directory forms a subsystem with a clear caller-facing interface, keep its implementation submodules private and re-export the interface from `mod.rs` via `pub use self::...;`. Import that interface through the subsystem root both inside and outside the subsystem, and import implementation-only details directly from their defining modules within the subsystem. Do not add a facade to a directory that merely groups unrelated modules, and do not forward the same symbol through multiple ancestors or expose duplicate import paths.
- **Conditional compilation**: Attach `#[cfg(...)]` attributes at `mod` boundaries whenever possible instead of repeating identical feature or target gates across individual items inside a file.

```rust
//! Binary framing and compression codecs.

use std::io;

use bytes::BytesMut;

use crate::error::Error;

pub mod frame;

mod compress;
pub use self::compress::Codec;
pub use self::compress::Compression;

mod buffer;
mod varint;

#[cfg(test)]
mod tests;
```

## Imports and Qualification

Configure `rustfmt.toml` so import grouping and doc-comment formatting stay deterministic across contributors and tools:

```toml
comment_width = 120
format_code_in_doc_comments = true
group_imports = "StdExternalCrate"
imports_granularity = "Item"
wrap_comments = true
```

- **Module-header imports by default**: Place shared `use` imports at the top of the enclosing module rather than repeating identical imports across functions. Keep a `use` local to a function, block, or `match` arm when scoping avoids trait method ambiguity (such as bringing an internal trait into scope for one generic helper without colliding with another trait imported across the file), isolates heavy helper-only namespaces (such as CLI styling, table formatting, snippet rendering, or large enum variant globs) to the single function that uses them, or keeps imports inside a `#[cfg(...)]` or `#[test]` item instead of scattering extra `#[cfg(...)]` attributes across top-of-file imports or wrapping a single `#[test]` in an artificial module.
- **No inline `crate::` or `super::` paths**: Never spell `crate::...`, `super::...`, or `self::...` inline in expressions or type signatures. Always bring intra-crate items into scope via `use crate::...` (at the module header or in a scoped local `use`), avoid `use super::...` and private `use self::...` imports so moving files across directories does not break relative paths, and reserve `use super::*;` exclusively for `#[cfg(test)]` and colocated benchmark modules.
- **External and standard-library inline paths when simpler**: Spelling out `std::...`, `core::...`, or `<external_crate>::...` inline is welcome whenever it reads more simply than adding an import, such as one-off calls (`std::env::var`, `std::panic::catch_unwind`), avoiding `#[cfg(...)]`-gated imports (`std::time::Duration` inside a `#[cfg(test)]` helper), or disambiguating colliding names (`impl core::error::Error for Error`, `jiff::Timestamp` vs. `proto::Timestamp`, or wrapper fields `Mutex(std::sync::Mutex<T>)`).
- **Meaningful qualification and minimal aliases**: Preserve original symbol names with enough module or crate qualification to clarify context and disambiguate common names (`io::Error`, `fs::read`, `process::Command`, `atomic::Ordering`), even without a local conflict. Avoid arbitrary `use ... as ...` renames because aliases obscure symbol origin and hurt workspace search; resolve collisions through parent-module qualification or external inline paths first, and import method-only traits with `use ...::Trait as _;` (at module scope unless local scoping avoids ambiguity or `#[cfg]` noise).

```rust
// Good: module-header qualified imports, scoped local `use`, and simple inline `std::` paths.
use std::fs;
use std::io;
use std::io::Write as _;
use std::ptr;
use std::str;

use crate::codec::Codec;

fn read_magic(path: &str, codec: &Codec, raw: *const u32) -> Result<u32, io::Error> {
    let bytes = codec.decode(&fs::read(path)?)?;
    let _header = str::from_utf8(&bytes).map_err(io::Error::other)?;
    io::sink().write_all(&bytes)?;
    // SAFETY: caller guarantees `raw` is valid for a 4-byte unaligned read.
    Ok(unsafe { ptr::read_unaligned(raw) })
}

#[test]
fn roundtrip_smoke() {
    use crate::codec::Header;

    let timeout = std::time::Duration::from_secs(1);
    let _ = (Header::default(), timeout);
}
```

```rust
// Bad: bare generic names lose context, `super::` is fragile, aliases hurt search,
// `#[cfg]` gates clutter file headers, and `crate::` paths appear inline.
use std::fs::read;
use std::io::Error as IoError;
use std::ptr::read_unaligned;
use std::str::from_utf8;
#[cfg(test)]
use std::time::Duration;

use super::codec::Header;
#[cfg(test)]
use super::codec::Fixture;

fn decode_bytes(bytes: &[u8]) -> Result<Vec<u8>, IoError> {
    crate::codec::Codec::decode(bytes)
}
```

## Visibility

Use a three-tier visibility model: private (the default), plain `pub`, and `pub(crate)`. Avoid `pub(super)` and `pub(in ...)`.

- **Private by default**: Keep items private when they are used only within their defining module and its descendants.
- **Module-bounded `pub`**: Let the enclosing module boundary define the visibility ceiling. Inside a module tree hidden behind a private ancestor (`mod internal;`), use plain `pub` for types, fields, and functions shared across modules in that tree rather than repeating `pub(crate)`.
- **Targeted `pub(crate)`**: Reserve `pub(crate)` for internal fields, constructors, or helpers that sit inside a public module (`pub mod`) or on a type publicly re-exported via `pub use`, where plain `pub` would leak them into the public API.
- **Test encapsulation**: Never widen production visibility solely to let another module's tests inspect internal details.

```rust
// src/lib.rs
mod session;
pub use self::session::Session;

mod storage; // private subsystem

// src/storage/page.rs (inside private `mod storage`)
pub struct Slot {
    pub offset: u64,
    pub len: u32,
}

// src/session.rs (`Session` is publicly re-exported at the crate root)
pub struct Session {
    id: u64,
}

impl Session {
    pub fn id(&self) -> u64 {
        self.id
    }

    // Reachable through public `Session`; restrict crate-internal construction with `pub(crate)`.
    pub(crate) fn from_raw(id: u64) -> Self {
        Self { id }
    }
}
```

## Types, Traits, and Attributes

- **Naming and layout**: Name types, fields, and functions by their concrete role, using the shortest name that stays clear. Keep a type and its primary `impl` block together, placing public API methods before private helpers. Reserve `len` and `is_empty` for collection or slice-like views; choose specific names when reporting byte size, capacity, weight, or retained samples.
- **Constructors and builders**: Use `new` for the primary constructor (implementing `Default` when a meaningful parameterless value exists, and marking trivial constructors `const fn` when practical) and `with_*` for alternative required inputs. Introduce a dedicated `*Builder` type when construction has multiple optional settings or staged validation rather than proliferating combinatoric constructors.
- **No leading underscores on fields**: Do not prefix struct, union, or enum-variant fields with `_` merely to silence unused-field warnings, as leading underscores obscure construction sites and bypass stale-lint checks. Name retained fields after the resource, guard, or marker they hold, and annotate unread ownership fields with `#[expect(dead_code, reason = "...")]` (`PhantomData` fields do not trigger `dead_code` and need neither `_` nor an attribute). By contrast, `_param` for unused function parameters and `let _guard = ...;` for local RAII drop guards remain idiomatic Rust.

```rust
use std::marker::PhantomData;
use std::sync::Arc;

// Good: descriptive field names and explicit ownership intent.
pub struct PooledConn<'a> {
    raw: RawConn,
    #[expect(dead_code, reason = "retains pool lease until the connection is dropped")]
    lease: Arc<Lease>,
    marker: PhantomData<&'a ()>,
}

// Bad: leading underscores obscure ownership and bypass stale-lint checks.
pub struct PooledConn<'a> {
    raw: RawConn,
    _lease: Arc<Lease>,
    _marker: PhantomData<&'a ()>,
}
```

- **Standard traits**: Derive applicable value traits (`Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`) on plain data types and configuration enums. Deliberately omit `PartialEq` and `Eq` when field-wise equality would reflect internal layout, history, or random seeds rather than a sound domain equality contract. Implement `Debug` manually on custom iterators and view types when `#[derive(Debug)]` would impose unnecessary generic bounds on type parameters.
- **Restrained `#[must_use]`**: Omit routine `#[must_use]` annotations on ordinary getters, queries, and consuming builder methods. Reserve `#[must_use = "..."]` with an explicit reason for lazy iterators or futures, RAII guards, non-`Result` outcome or report types whose dismissal silently loses failures or resources, and `&self` methods that callers could mistake for in-place mutators.
- **Lint baseline and `#[expect]`**: Prefer `#[expect(lint, reason = "...")]` over `#[allow(lint)]` so local suppressions fail once the underlying warning disappears; targeted `#[allow(lint)]` with an explanatory comment is acceptable only when reachability genuinely varies across feature or toolchain matrices. In `Cargo.toml`, configure workspace lints to catch unknown lints, ignored `must_use` values, stray debug macros, and unintended `unsafe` usage:

```toml
[workspace.lints.rust]
unknown_lints = "deny"
unsafe_code = "deny"          # Use `unsafe_op_in_unsafe_fn = "deny"` in crates that require unsafe code
unused_must_use = "deny"

[workspace.lints.clippy]
dbg_macro = "deny"
```

## Control Flow and Error Semantics

- **Linear control flow**: Keep abstractions at function signatures, cohesive types, and module boundaries. Inside functions, write direct, top-to-bottom logic using early guard returns and explicit `match` or `if let` branches.
- **Loops vs. combinators**: Prefer explicit `for` loops for multi-step transformations, stateful accumulation, or fallible validation; use iterator chains for concise, side-effect-free mappings.
- **Earned complexity**: Require every helper, wrapper type, or trait layer to justify its cognitive cost. Small local duplication that keeps branches independent and readable is preferable to premature indirection or defensive glue code without an explicit contract.
- **Errors, absence, and panics**:
  - Return `Result<T, E>` for recoverable failures that callers can handle, report, or retry. Keep failing mutations atomic with respect to observable `self` state whenever practical.
  - Return a natural empty value (`0`, `false`, or an empty collection) or `Option<T>` for absent query results rather than modeling absence as an error.
  - Fail fast (`assert!`, `panic!`, `unreachable!`) on violated internal invariants or unrecoverable programming bugs instead of widening infallible signatures to `Result` or silently substituting fallback values.
  - Preserve structured causes and domain context across error boundaries; expose typed fields or accessors when callers need programmatic recovery rather than forcing string inspection on formatted error messages.

## Public API Documentation and Comments

- **Observable contracts only**: Write `///` documentation from the caller's perspective, covering behavior, preconditions, ownership transfers, and state transitions. Keep internal data structures, lock mechanics, trait-coherence workarounds, and historical design rationale out of public API docs.
- **Voice and formatting**: Describe types with noun phrases (`A bounded frame decoder.`) and functions or methods with third-person present-tense verbs (`Creates`, `Decodes`, `Returns`). End summary sentences with periods, and wrap Rust identifiers, literals, and numeric ranges in inline code spans.
- **Clean intra-doc links**: Use public API names as intra-doc link labels and hide private module paths in explicit link targets (such as `` [`Codec`][crate::compress::Codec] ``) so internal module layouts do not leak into rendered prose.
- **Contract sections**: Include section headings only when they state an actual contract, ordered before `# Examples`:
  - Explain straightforward return values in the summary or body; add an optional `# Returns` section only for non-trivial outcomes, snapshots, or ownership rules.
  - `# Errors`: Concrete conditions under which a fallible operation returns `Err`.
  - `# Panics`: Concrete caller-visible panic preconditions.
  - `# Safety`: Invariants callers must uphold when calling an `unsafe fn` or implementing an `unsafe trait`.
  - Domain-specific contract sections (when a crate defines them) appear before `# Examples`.
  - `# Examples` (optional, always last): Demonstrate realistic behavior rather than mechanical syntax, keeping `use` imports visible (a trailing hidden `Ok::<(), Error>(())` is fine to support `?`).
- **Inline comments**: Prefer self-explanatory code over commentary that restates what the code does. Reserve `//` comments for non-obvious invariants, concurrency or `unsafe` preconditions (`// SAFETY: ...`), external protocol quirks, or performance rationale that types and names cannot express.

## Testing

- **Contract-first coverage**: Exercise observable behavior through public APIs whenever practical; keep `#[cfg(test)]` unit tests next to the implementation strictly for non-trivial private invariants or state machines that black-box tests cannot cover cleanly.
- **High-value tests only**: Every test should protect against a plausible bug or boundary regression without inflating maintenance cost. Do not add tests merely because code changed, and remove tests that restate literals, mirror trivial control flow, check static type guarantees, or assert non-contract `Debug` or diagnostic wording.
- **Production purity and determinism**: Do not add test-only hooks or complicate production paths to expose internal state. In concurrent tests, coordinate execution deterministically (with channels, barriers, notifications, or controlled clocks) instead of sleeping.

## Integrating into `AGENTS.md`

See [AGENTS.template.md](AGENTS.template.md) for a condensed, example-free version ready to copy or adapt into a repository's `AGENTS.md` alongside project-specific architecture and domain contracts.
