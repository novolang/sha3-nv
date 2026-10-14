# Changelog

All notable changes to sha3-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-10-11

**The implementation, published over the interface.** Every body is
written and every suite is green: Keccak-p[1600, 24], the sponge over
it, the four fixed-width hashes, SHAKE128 and SHAKE256 with resumable
squeezing, and cSHAKE and KMAC from SP 800-185.

### Added

- The bodies: `sha3keccak`'s five step mappings and its twenty-four
  rounds; `sha3sponge`'s byte-at-a-time absorb and squeeze over a
  `@value` state; the four widths as one parameterised state; the two
  SHAKEs; and SP 800-185's length encodings, cSHAKE and KMAC.
- `tests/sha3_edge_tests.nv` — the supplementary suite. It
  re-derives FIPS 202's three tables inside the test (the round
  constants from Algorithm 5's LFSR and Algorithm 6's assembly, the
  rho offsets from Algorithm 2's walk, and pi checked as a permutation
  against the § 3.2.3 statement), pins every refusal's message, code
  and predicate, and carries samples the vector suite does not:
  SHA3-256 and SHA3-512 either side of the rate boundary, 200 bytes of
  `0xa3` at all six functions, SHAKE128's 168- and 169-byte outputs,
  cSHAKE128 and cSHAKE256 sample 2, KMAC256 samples 1 to 3, and
  KMACXOF128 samples 1 to 3 plus KMACXOF256 sample 1.

### Changed

These break code written against 0.0.x.

- Thirteen public functions now take their output buffer as a `var`
  parameter, because a buffer a function fills in place must be
  reachable through a `var` name: `sha3hash.finish` and `digest_into`;
  `sha3xof.xof_squeeze`, `shake128_into` and `shake256_into`; and
  `sha3kmac.cshake128_into`, `cshake256_into`, `kmac_finish`,
  `kmac_finish_xof`, `kmac128`, `kmac256`, `kmac128_xof` and
  `kmac256_xof`.  A caller passes a `var` name or a new buffer.  `var`
  is the shape the language leaves: a plain buffer parameter cannot be
  written through (E2036) and a `var` one cannot take a `let`-bound
  buffer (E2038).  Filed as
  `language/a-frozen-into-signature-cannot-fill-its-buffer`; put the
  plain signatures back when that filing closes.
- `sha3xof.Sha3Xof` has one more field, `squeezed`, the number of
  output bytes the state has produced; `xof_squeezed` reads it, and a
  struct literal of `Sha3Xof` needs the new field.
- Six bindings in four fixtures of the vector, cover and kmac suites
  bind an output buffer with `var` instead of `let`, which is the only
  binding the `var` parameters above accept (E2038); the
  `sha3xof.xof_squeeze` doc example does the same for its two 16-byte
  buffers.  No assertion changed: `git diff tests/` is those six
  bindings and the deleted release stamps.

### Known

- **`Sha3AbsorbAfterSqueeze` is published and nothing in this package
  raises it.**  `sha3hash.update` and `sha3xof.xof_update` answer a
  state rather than a `Result`, and a byte fed to a sealed sponge is
  ignored rather than refused, so there is nowhere for the refusal to
  come from.  `is_sealed` and `xof_is_sealed` are the question to ask
  first; the arm is the code for a caller that reports the mistake
  itself.  README rule 12, `sha3sponge.absorb_byte`'s doc comment and
  the README's paragraph on the low-level interface used to say
  otherwise and were corrected.
- **`sha3hash.is_finished` answers `false` for every state a caller can
  hold.**  `finish` seals a copy and `update` cannot seal a state, so
  no published call hands back a finished `Sha3State`; the function is
  the same question `sha3sponge.is_sealed` asks of the sponge.  Its doc
  comment used to promise an `update` refusal that cannot happen.
- **`Sha3RateNotPositive` is published and nothing in this package
  raises it either.**  Every rate this package uses is a named
  constant, and `sha3sponge.sponge` takes a rate as a number, so the
  arm is the code for a caller that computes a rate of its own.  The
  header of `sha3err` said a rate of zero "would divide by zero in the
  implementation", and `sha3sponge.sponge`'s doc comment promised a
  check above it; nothing divides by the rate and no published call
  passes one, and both sentences now say what is true.
- **The sponge suite cites "Appendix B" for the round constants.**  In
  FIPS 202 that appendix holds the conversion functions and the
  hexadecimal padding forms; the constants are computed by § 3.2.5's
  Algorithms 5 and 6 and are not tabulated in the standard at all.  The
  suite is the oracle and is not edited, so the sentence (its header
  and its first round-constant case) stands and is recorded here; the
  same citation in `sha3keccak`'s doc comments was corrected.

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
