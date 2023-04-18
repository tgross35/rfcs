- Feature Name: feature-metadata
- Start Date: 2023-04-14
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/3416)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

[summary]: #summary

The main purpose of this RFC is to define a structured way to document and add
attributes to Cargo features in `Cargo.toml`. This will be usable by `rustdoc`
for user-facing feature descriptions, as well as by Cargo itself for build
related user output.

This RFC only describes a new `Cargo.toml` schema; `rustdoc`'s handling of this
information is in a separate RFC.

# Motivation

[motivation]: #motivation

Features are widely used as a way to do things like reduce dependency count,
gate std or alloc-dependent parts of code, or hide unstable API. Use is so
common that many larger crates wind up with tens of feature gates, such as
[`tokio`] with 24. Despite being a first class component of crate structure,
there are some problems that don't have elegant solutions:

- Documentation is difficult, often requiring library authors to manually manage
  a table of descriptions
- There is no way to deprecate features or hide unneeded features

This RFC proposes a plan that solves these problems.

[`tokio`]: https://docs.rs/crate/tokio/latest/features

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Usage is simple: features will be able to be specified in a table (inline or
separate). Sample section of `Cargo.toml`:

```toml
[features]
# current configuration will continue to work
foo = []
# New configurations
bar = { requires = ["foo"], doc = "simple docstring here"}
baz = { requires = ["foo"], public = false, unstable = true }
qux = { requires = [], deprecated = true }
quux = { requires = [], deprecated = { since = "1.2.3", note = "don't use this!" } }

# Features can also be full tables if descriptions are longer
[features.corge]
requires = ["bar", "baz"]
doc = """
# corge

This could be a longer description of this feature
"""
```

The following keys would be allowed in a feature object:

- `requires`: This is synonymous with the existing table describing required
  features. For example, `foo = ["dep:serde", "otherfeat"]` will be identical to
  `foo = { requires = ["dep:serde", "otherfeat"] }`
- `doc`: A markdown docstring describing the feature. Like with `#[doc(...)]`,
  the first line will be treated as a summary.
- `public`: A boolean key defaulting to `true` that indicates whether or not
  downstream crates should be allowed to use this feature.
- `deprecated`: This can be either a simple boolean, or an object with `since`
  and/or `note` keys. Cargo will warn downstream crates using this feature,
  similar to the [`deprecated`] attribute.
- `unstable`: This flag indicates that the feature is only usable with nightly
  Rust

[`deprecated`]: https://doc.rust-lang.org/reference/attributes/diagnostics.html#the-deprecated-attribute

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

Validation and parsing of the new schema, described above, shoud be relatively
straightforward. Depending on the exact semantics decided upon

A possible future option is to allow giving the feature documentation in a
separate file, allowing a markdown section specifier.

```toml
foo = { requires = [], doc-file = "features.md#foo" }
bar = { requires = [], doc-file = "features.md#bar" }
```

# Drawbacks

[drawbacks]: #drawbacks

- Added complexity to Cargo. Parsing is trivial, but 
- Docstrings can be lengthy, adding noise to `Cargo.toml`. This could be solved
  with the above mentioned `doc-file` key.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- TOML-docstrings (`## Some doc comment`) and attributes that mimic Rust
  docstrings could be used instead of a `doc` key. This is discussed in
  [future-possibilities], but decided against to start since it is much simpler
  to parse standard TOML.
- Feature descriptions could be specified somewhere in Rust source files. This
  has the downside of creating multiple sources of truth on features.

# Prior art

[prior-art]: #prior-art

- There is an existing crate that uses TOML comments to create a features table:
  <https://docs.rs/document-features/latest/document_features/>
- `docs.rs` displays a feature table, but it is fairly limited
- Ivy has a [visibility attribute] for its configuration (mentioned in [cargo #10882])

[visibility attribute]: https://ant.apache.org/ivy/history/latest-milestone/ivyfile/conf.html
[cargo #10882]: https://github.com/rust-lang/cargo/issues/10882

# Unresolved questions

[unresolved-questions]: #unresolved-questions

RFC-blocking questions:

- Do we want all the proposed keys? Specifically, `public` and `unstable` may be
  more than what is needed.

  See also:
  - <https://github.com/rust-lang/cargo/issues/10882>
  - <https://github.com/rust-lang/cargo/issues/10881>

- What should the exact semantics of these keys be?
- Does it make sense to have separate `hidden` (not documented) and `public`
  (feature not allowed downstream) attribute?

It is worth noting that simpler keys (`requires`, `doc`, `deprecated`) could be
stabilized immediately and other features could be postponed.

# Future possibilities

[future-possibilities]: #future-possibilities

- Rustdoc would gain the ability to document features. This is planned in an
  associated RFC.
- Cargo could parse doc comments in `Cargo.toml`, like the above linked
  `document-features` crate. This would adds complexity to TOML parsing.

  ```toml
  [features]
  foo = { requires = [], doc = "foo feature" }
  ## foo feature
  foo = []
  ```

- `unstable` or `nightly` attributes for features could provide further
  informations or restriction on use (see
  <https://github.com/rust-lang/cargo/issues/10881>)
