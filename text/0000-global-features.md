- Feature Name: `global-features`
- Start Date: 2023-09-20
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Cargo [features](https://doc.rust-lang.org/cargo/reference/features.html)
are targeted at direct dependents while some decisions need to be made by the top-level crate (usually a `[[bin]]`).
This is currently worked around by tunneling features up or relying on environment variables.

This RFC proposes an alternative to features called `globals` that are targeted
at decisions that affect the entire final artifact.

# Motivation
[motivation]: #motivation

Use cases
- An crate author offers optional optimizations, like using `parking_lot`
  - Currently solved by using a feature like in
    [tokio](https://github.com/tokio-rs/tokio/blob/ad7f988da377c365cacb5ca24d044a9be5de5889/tokio/Cargo.toml#L100)
  - This requires callers to either enable it directly, re-export the feature, or have applications directly depend on `tokio` and enable it.
- `-sys` crates need to make the decision of whether to statically link a vendored version of the source of dynamically link against a system library.
  - Currently this is solved by a variety of means, usually by dynamically linking if the system library is available and then falling back to the vendored copy unless a vendored feature is enabled.
  - This requires callers to either enable it directly, re-export the feature, or have applications directly depend on the `-sys` crate and enable it.
  - To properly represent this, we need three states: "dynamic", "static", "auto"
  - See also [Internals: Pre-RFC Cargo features for configuring sys crates](https://internals.rust-lang.org/t/pre-rfc-cargo-features-for-configuring-sys-crates/12431)
- Enable `alloc` or `std` features across the stack ([rust-lang/cargo#2593](https://github.com/rust-lang/cargo/issues/2593#issuecomment-220474831))
- Multiple backends
  - flate2 has [multiple backends](https://github.com/rust-lang/flate2-rs/blob/f62ff42615861f1d890160c5f77647466505eac0/Cargo.toml#L33C1-L33C1)
  - Inkwell supports [multiple LLVM backends](https://github.com/TheDan64/inkwell)
  - [mlua](https://github.com/khvzak/mlua) supports multiple backends and Lua versions
  - async-std vs tokio runtimes
- Module-level parameters
  - Inline-size control in [kstring](https://docs.rs/crate/kstring/2.0.0/features)

Why are we doing this? What use cases does it support? What is the expected outcome?

See also
- [rust-lang/cargo#9094](https://github.com/rust-lang/cargo/issues/9094)
- [rust-lang/cargo#1555](https://github.com/rust-lang/cargo/issues/1555)
- [rust-lang/cargo#2980](https://github.com/rust-lang/cargo/issues/2980)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`Cargo.toml` gains a new `[globals]` section
```toml
[package]
name = "foo"
version = "0.0.0"

[globals.sys]
description = "External library link method"
values = ["static", "dynamic", "auto"]
self-default = "auto"

[target.'cfg(any(global::sys = "dynamic", global::sys = "auto"))'.build-dependencies]
pkg_config = "0.3.27"
[target.'cfg(any(global::sys = "static", global::sys = "auto"))'.build-dependencies]
cc = "1.0.83"
```
- Global names follow the same naming rules as cargo features
- Global values may be any UnicodeXID continue character

These can then be referenced in the source, like this `build.rs`
```rust
#[cfg(any(global::sys = "dynamic", global::sys = "auto"))]
fn dynamic() -> Option<Target> {
    // ...
}

#[cfg(any(global::sys = "static", global::sys = "auto"))]
fn static() -> Option<Target> {
    // ...
}

fn auto() -> Option<Target> {
    #[cfg(any(global::sys = "dynamic", global::sys = "auto"))]
    if let Some(target) = dynamic() {
        return Some(target);
    }

    #[cfg(any(global::sys = "static", global::sys = "auto"))]
    if let Some(target) = static() {
        return Some(target);
    }

    None
}
```
- References by this crate to global names/values in `cfg`s and `cargo:rustc-cfg` build script directive will be validated based on
  [RFC 3013](https://rust-lang.github.io/rfcs/3013-conditional-compilation-checking.html) (rustc and cargo)

Workspace globals may also be specified and a package may inherit them and append, but not modify, them:
```toml
[workspace.globals.sys]
description = "External library link method"
values.enum = ["static", "dynamic", "auto"]
self-default = "auto"
```
```toml
[globals]
workspace = true

[globals.zlib]
description = "zlib implementation"
values = ["miniz", "zlib", "zlib-ng"]
set-local = "auto"
```
`cargo new` will not automatically inherit `[workspace.globals]`, like it does for `[workspace.package]` and `[workspace.lints]`.

Top-level callers may configure globals
```toml
[package]
set-globals = [ "sys=static", "zlib="miniz" ]
# may also be inherited from the workspace
# set-globals.workspace = true
```
Also
- On the command-line as `--globals sys=dynamic`
- As a config setting

Instead of unifying `set-globals`, it is a compilation error when multiple top-level crates are being built at once with different `set-globals`.

On `cargo publish`, `workspace.set-globals` is copied to `package.set-globals` to apply to `cargo install`

When resolving dependencies (the feature pass), fingerprinting of packages, an  when compiling the packages,
`workspace.set-globals` will only apply to a package if the value is valid within the schema for that package,
otherwise the default will be used if any.
Any unused globals will be a compilation error.

`cargo <cmd> --globals` (no argument) will report the globals available for
being set within the current package along with their valid values.

## SemVer

SemVer Compatible
- Adding a global
- Adding a global value

Semver Incompatible
- Removing a global
- Removing a global value

Context-dependent
- Changing a global default

# Drawbacks
[drawbacks]: #drawbacks

TODO

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This couples mutually exclusive features with global features because
- Mutually exclusive features provides design insight into what we should do for global features
- Scoping mutually exclusive features to global features solves the feature-unification problem

Global features take a value, rather than allowing relying on presence like normal `cfg`s
(see [RFC 3013](https://rust-lang.github.io/rfcs/3013-conditional-compilation-checking.html))
- Less brittle for future evolution
- Easier to unify for a distributed schema
- Removes the complexity in declaratively defining presence and value instances

`package.set-globals` is a package setting, with conflict errors, rather than a
workspace setting because people will likely have binaries that serve very
different roles within the same workspace (different target platforms, etc).

`package.set-globals` is an array of strings in case we add multi-valued globals in the future.

## Alternatives

### Native support for controlling `std` / `alloc` / `core`

See [Internals: Pre-Pre-RFC: making `std`-dependent Cargo features a first-class concept](https://internals.rust-lang.org/t/pre-pre-rfc-making-std-dependent-cargo-features-a-first-class-concept/10828)

### Native cargo support for `-sys` crates

Instead of a convention around global features, cargo could have built-in flags
for controlling `-sys` crates and could ship some native support that would
remove some boilerplate and ensure consistency.

Downsides
- Longer time frame: this would require more investigation and experimentation.
- Doesn't handle all use cases

See
- [Internals: Direct support for pkg-config](https://internals.rust-lang.org/t/direct-support-for-pkg-config-in-cargo/4411)

### Automatic feature activation

Too many spooky effects

See
- [RFC #1787](https://github.com/rust-lang/rfcs/pull/1787)
- [rust-lang/cargo#1555](https://github.com/rust-lang/cargo/issues/1555)
- [rust-lang/cargo#2593](https://github.com/rust-lang/cargo/issues/2593#issuecomment-212508225)

### Mutually exclusive features conflict via an identifier

Some systems call this "provides" and others a "capablity".

Downsides
- No way to unify these, the decision needs to be at the top-level crate.
- Linux distributions that use this implicitly wire the packages together by
  dropping files in place with compatible names while Rust/Cargo need explicit
  wiring, requiring the choice to be bubbled up the stack.  See instead "facade crates".

See
- [Pre-RFC cargo-mutex features](https://github.com/ctron/rfcs/blob/feature/cargo_feature_caps_1/text/0000-cargo-mutex-features.md)
- [Internals: Pre-RFC Cargo mutually exclusive features](https://internals.rust-lang.org/t/pre-rfc-cargo-mutually-exclusive-features/13182)

### Mutually exclusive features that instantiate two copies of a dependency

While there might be a place for a variant of this idea, most motivating cases
are dealing with wanting one version of the dependency.

See [rust-lang/cargo#2980](https://github.com/rust-lang/cargo/issues/2980#issuecomment-667207117)

See also [Internals: module-level generics](https://internals.rust-lang.org/t/pre-rfc-module-level-generics/12015)

See also [Cabal backpack support](https://cabal.readthedocs.io/en/stable/cabal-package.html#backpack)

### `log` / logger split

Some cases can be modeled like `log` where a trait is defined and an API is
exposed for regisering a specific implementation.

If the interface crate is able to also depend on the implementations, it could
also initialize the global with a default on first use if its unset.

Benefits
- Works today
- Flexible on what state an implementation can be initialized with

Downsides
- Doesn't work for all use cases
- Imperative wiring together of APIs in the top-level crate
- Some level of overhead
- Supporting a default requires building all implementations since the choice is at runtime

### Facade crates

On [internals](https://internals.rust-lang.org/t/idea-facade-crates/18051),
the idea was proposed to bake-in support for the log pattern so it can be done at compile time.

Downsides
- Doesn't work for all use cases like `-sys` crates

# Prior art
[prior-art]: #prior-art

## Gentoo USE flags

- [Top-level documentation](https://wiki.gentoo.org/wiki/USE_flag)
  - [Using USE flags](https://wiki.gentoo.org/wiki/Handbook:AMD64/Working/USE#Using_USE_flags)
- [index of all USE flags](https://www.gentoo.org/support/use-flags/)

USE flags are booleans (set or unset) that are defined through a form of layered config,
with some defaulted to on.
Users may then enable more or disable some of the defaulted flags.
This can be done globally or on a per-package basis.
Packages then check what is enabled and conditionalize their builds off of them.

## Bazel

[Features](https://docs.bazel.build/versions/0.26.0/cc-toolchain-config-reference.html#features)
may "provide" a feature and it is an error to "provide" a feature more than once.

## Gradle

This seems work similar to Bazel but instead of "providing" a feature, you
[declare a capability](https://docs.gradle.org/current/userguide/feature_variants.html#sec::incompatible_variants).

See also https://docs.gradle.org/6.0.1/userguide/dependency_capability_conflict.html

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Naming
- Target-specific `set-globals`

# Future possibilities
[future-possibilities]: #future-possibilities

## `globals.*.values`

Potential variants for this tagged enum include:
- `globals.foo.values = "bool"`
- `globals.foo.values.range = "8-23"`
- `globals.foo.values.path = "ImplForTrait"`
- `globals.foo.values.expr = "some_func()"`

## Multi-valued globals

Extend the definition of a global to include a `multi` field.
Instead of new assignments overwriting, they append.

## `cfg_value!`

Expose the valid value for the specified global as a `&'static str`
```rust
let foo = cfg_str!("foo");
```