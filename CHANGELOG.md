# Changelog

All notable changes to sha3-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `sha3keccak` — Keccak-p[1600, 24], the one primitive the rest of the
  package is made of. The 1600-bit state is a `@value` struct holding
  twenty-five 64-bit lanes, so it lives in the caller's own stack
  frame with no header, no reference count and nothing to free. The
  five step mappings are published separately from the round that
  composes them, because a failing implementation is bisected a step
  at a time against the intermediate states the Keccak team publish.
- `sha3sponge` — the sponge, and the decision the whole package rests
  on: a sponge IS a rate and a domain-separation byte, and those two
  numbers are the entire difference between SHA3-256, SHAKE256,
  cSHAKE256 and KMAC256. The table is in the module's own comment. A
  byte-at-a-time absorb and a byte-at-a-time squeeze, so a device
  hashes with no buffer at all.
- `sha3hash` — the four fixed widths as ONE state parameterised by a
  `Sha3Width`, rather than four near-identical families. blake2-nv
  publishes two families because BLAKE2b and BLAKE2s really are two
  algorithms with different word sizes; SHA3-224 through SHA3-512 are
  one algorithm with four rates. `finish` writes into a buffer the
  caller owns and refuses one of the wrong length rather than
  truncating.
- `sha3xof` — SHAKE128 and SHAKE256, with squeezing that resumes.
  `xof_squeeze` answers the state positioned after the bytes it wrote,
  so a key stream is taken in pieces and the pieces concatenate. There
  is no `SHAKE128_DIGEST_BYTES`, because the number in the name is a
  security strength and not a length.
- `sha3kmac` — SP 800-185's three length encodings, cSHAKE and KMAC.
  The output length is deliberately NOT a field of `Sha3Kmac`:
  `right_encode(L)` is appended to the message at the end, so
  `kmac_finish` learns the length from the buffer it is handed, at the
  one moment it can still be encoded.
- `sha3err` — four refusals, with `is_length_fault` and
  `is_order_fault` dividing "I sized a buffer wrong" from "my state
  machine is wrong", which are different bugs and different fixes.
- `tests/embedded_probe.nv` — the device claim as a program. It builds
  a Cortex-M4 executable from `sha3keccak` and `sha3sponge` and links.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the four API suites reaches `not implemented:
  sha3-nv.<module>.<fn>`.
- **`xof_squeeze` answers the state and not also the buffer.** A
  `@value` struct may not be a tuple element (E2015), so
  `(Sha3Xof, Bytes)` is refused. The buffer is the caller's own and
  the caller still holds it, so nothing is lost; the signature says so
  rather than boxing the state to make a pair possible.
- **No dependency on crypto-nv.** Checking a KMAC tag needs a
  constant-time comparison and the registry's is `digest.ct_eq` in
  crypto-nv. Depending on it for that one function would put SHA-2,
  SHA-1, MD5 and HMAC into the footprint of a device that wanted SHA-3
  and nothing else. blake2-nv made the same call; the README says
  where the comparison lives.
- **TupleHash and ParallelHash are named as missing**, not stubbed.
  SP 800-185 defines them on cSHAKE and nothing on the registry needs
  them yet.
