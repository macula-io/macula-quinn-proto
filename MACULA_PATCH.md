# Macula fork of `quinn-proto` 0.11.14

Vendored fork of [quinn-rs/quinn](https://github.com/quinn-rs/quinn)'s
`quinn-proto` sub-crate at version `0.11.14`, with two single-line patches
removing `features = ["std"]` from the `rustls` and `tracing`
dependency declarations.

Used by [macula-kernel](https://codeberg.org/macula-internal/macula-kernel)
as the in-kernel QUIC state machine. The kernel target is `no_std` and
cannot satisfy `std`-feature demands forced through unification.

## The patch

In `Cargo.toml` (cargo-normalised; the form cargo actually consumes):

```diff
 [dependencies.rustls]
 version = "0.23.5"
-features = ["std"]
 optional = true
 default-features = false

 [dependencies.tracing]
 version = "0.1.10"
-features = ["std"]
 default-features = false
```

`Cargo.toml.orig` is unchanged because in the upstream workspace these
features are set at the workspace level, not at the consuming crate.
Cargo reads `Cargo.toml` (not `Cargo.toml.orig`) when consuming a crate
via git source.

## Why this is needed

Upstream quinn-proto unconditionally enables `rustls/std` and `tracing/std`
on its own deps, even when the consumer requests
`quinn-proto = { default-features = false, features = ["rustls-ring"] }`.

That cascade does the damage:

- `rustls/std` propagates `std` to `once_cell`, `webpki`, and
  `rustls-pki-types`, none of which can compile against an `os = "none"`
  target with `std` on (E0463: can't find crate for `std`).
- `tracing/std` propagates `std` to `tracing-core`, which in turn
  re-enables `once_cell` default features, which re-enables
  `once_cell/std`, which fails the same way.

Result: 358 errors in `rustls-pki-types`, 245 in `once_cell`, plus
secondary failures in webpki and tracing-core. All eliminated by this
two-line patch.

All three downstream crates (`rustls`, `tracing`, `once_cell`) declare
`#![no_std]` or `#![cfg_attr(not(feature = "std"), no_std)]` at the top
of their source. They are fully no_std-capable when `std` is not
explicitly demanded. quinn-proto's hardcoded `features = ["std"]` is
what made this look intractable in the original scout report.

## What the patch does NOT change

- `default-features = false` stays.
- The `rustls-ring` cargo feature still enables `rustls?/ring` and
  `ring` (the crypto provider) — TLS plumbing is intact.
- The `tracing` dep still works for emitting trace events.
- No source-level changes to quinn-proto. The state machine, packet
  encoder, congestion control, etc. are byte-for-byte upstream.

## Upstream contribution

The intent is to file an upstream PR proposing one of:

1. Drop the `features = ["std"]` (simplest, but may break someone).
2. Add a quinn-proto cargo feature `std` that controls this propagation,
   so no_std consumers can opt out.

(2) is the better long-term fix. Open question whether upstream wants
to gate on this; the crate has not historically advertised no_std support.
If they accept, retire this fork.

## Versioning policy

Track quinn-proto upstream releases of 0.11.x. Re-roll by:

1. Downloading `quinn-proto-X.Y.Z.crate` tarball from crates.io.
2. Extracting into a fresh checkout.
3. Re-applying the two `features = ["std"]` removals to `Cargo.toml`.
4. Tagging as `vX.Y.Z-macula1`.
5. Bumping the git ref in `macula-kernel`'s `Cargo.toml`.

When the upstream PR lands (if it does), retire this fork.

## License

Inherited from upstream `quinn-proto` (Apache-2.0 OR MIT). No new code,
only the two Cargo.toml feature-list removals.
