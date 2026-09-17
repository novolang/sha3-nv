# sha3-nv

SHA-3 is a family of cryptographic hash functions, specified in
[FIPS 202](https://csrc.nist.gov/pubs/fips/202/final). The same
document specifies SHAKE128 and SHAKE256, which produce output of any
length the caller asks for.
[NIST SP 800-185](https://csrc.nist.gov/pubs/sp/800/185/final) builds
cSHAKE and KMAC on top of those. This package brings all of them to
novo-lang.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What SHA-3 is

A cryptographic hash turns a message of any length into a short,
fixed-length **digest**. SHA-3 is the family NIST standardised in 2015
after an open competition, and it is built on a different principle
from SHA-2.

SHA-2 is a **compression function** run over the message in blocks,
chaining one block's output into the next. SHA-3 is a **sponge**. A
sponge is three things: a permutation, a **rate** and a **padding
rule**. The permutation here is Keccak-p[1600, 24], which shuffles 200
bytes of **state** and has no message, key or length in it at all. The
state is split into the first `rate` bytes, which the message touches,
and the rest, the **capacity**, which it never touches. **Absorbing**
exclusive-ors message bytes into the rate and runs the permutation
whenever the rate fills. **Squeezing** reads bytes back out of the rate
and runs the permutation whenever it runs out.

The capacity is the security argument. An attacker who never sees those
bytes and never writes them cannot steer the state. FIPS 202 sets each
function's capacity to twice its security strength, which is what
decides its rate: the rate is 200 bytes minus the capacity.

A **domain-separation byte** is the one other thing that varies. It is
appended to the message as part of the padding, and it is what makes
SHA3-256 and SHAKE256 — which have the same rate and the same
permutation — unrelated functions of the same message.

| Function | Rate | Capacity | Domain byte | Output |
| --- | --- | --- | --- | --- |
| SHA3-224 | 144 | 448 bits | `0x06` | 28 bytes |
| SHA3-256 | 136 | 512 bits | `0x06` | 32 bytes |
| SHA3-384 | 104 | 768 bits | `0x06` | 48 bytes |
| SHA3-512 | 72 | 1024 bits | `0x06` | 64 bytes |
| SHAKE128 | 168 | 256 bits | `0x1f` | any length |
| SHAKE256 | 136 | 512 bits | `0x1f` | any length |
| cSHAKE128, KMAC128 | 168 | 256 bits | `0x04` | any length |
| cSHAKE256, KMAC256 | 136 | 512 bits | `0x04` | any length |

An **extendable-output function** (XOF) is a hash whose output length
the caller picks. SHAKE128 and SHAKE256 are the two FIPS 202 defines.
The number in the name is a security strength in bits, not an output
length: 64 bytes read out of SHAKE128 is 64 bytes with 128 bits of
security behind it.

**cSHAKE** is SHAKE with a **customization string** mixed in, so that
two protocols hashing the same message do not get the same bytes. A
**message authentication code** (MAC) is a hash under a secret key, so
that only a holder of the key can produce or check the value. **KMAC**
is the MAC SP 800-185 builds on cSHAKE.

## Install

```
novo pkg add sha3-nv
```

## Example

```novo
use std.bytes
use sha3hash
use sha3xof

fn main() [io]
    // The SHA3-256 digest of "abc", into a buffer this call allocates.
    println(bytes.to_hex(sha3hash.sha3_256(bytes.from_str("abc"))))
    // 3a985da74fe225b2045c172d6bd390bd855f086e3e9d525b46bfe24511431532

    // The same digest into a buffer you own, which is how a program
    // that hashes a thousand messages does it.
    let out = bytes.zeros(32)
    match sha3hash.digest_into(Sha3Width256, bytes.from_str("abc"), out)
        Ok(d)  => println(bytes.to_hex(d))
        Err(e) => println(e.message())

    // Thirty-two bytes of SHAKE128. Ask for sixty-four and the first
    // thirty-two are these.
    println(bytes.to_hex(sha3xof.shake128(bytes.from_str(""), 32)))
    // 7f9c2ba4e88f827d616045507605853ed73b8093f6efbc88eb1a6eacfa66ef26
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented:
sha3-nv.<module>.<fn>` panic. The tests are the specification the
implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `sha3keccak` | The Keccak-p[1600, 24] permutation: the 200-byte state as a value, the five step mappings, the round constants and the byte order. Uses no heap memory. |
| `sha3sponge` | The sponge over that permutation: a rate, a domain-separation byte, absorbing a byte at a time, the padding, and squeezing a byte at a time. Uses no heap memory. |
| `sha3hash` | The four fixed-width hashes over `Bytes`: one state parameterised by a width, the streaming form, the one-shot form, and the digest and rate constants. |
| `sha3xof` | SHAKE128 and SHAKE256: absorbing, and squeezing that can be resumed, into a buffer the caller supplies. |
| `sha3kmac` | SP 800-185: the three length encodings, cSHAKE, and KMAC in both its fixed-length and its extendable-output forms. |
| `sha3err` | The four refusals, with the question of whether each is about a length the caller chose or an order the caller used. |

## How to choose an entry point

**`sha3hash` is for a digest of a length somebody else fixed.** A
protocol that says SHA3-256 means 32 bytes and nothing else. The
one-shot functions `sha3_224`, `sha3_256`, `sha3_384` and `sha3_512`
allocate the result; `digest_into` and `finish` write into a buffer you
already own.

**`sha3xof` is for output of a length you choose.** A key stream, a
mask, a seed expander, or a digest at a size the protocol names itself.
Squeezing is resumable, so a caller can take the output in pieces.

**`sha3kmac` is for a keyed tag, or for a hash that must not collide
with another protocol's.** KMAC for the tag, cSHAKE for the
customization string.

**Firmware calls `sha3keccak` or `sha3sponge` directly.** These two
modules use no `Bytes`, no strings and no lists. A sponge is a value
that lives on the caller's own stack: the 200-byte state and four small
fields. You feed the message one byte at a time, seal it, and read the
output one byte at a time. Nothing is allocated from start to end. See
"Running on a microcontroller".

## The rules a user needs

1. **A narrower SHA-3 digest is not the front of a wider one.** The
   four widths have four different rates, so SHA3-224 of a message and
   SHA3-256 of the same message share no bytes. FIPS 202 section 6.1.
2. **A SHAKE output at any length is the front of every longer one.**
   No length is mixed into the state, so the first 32 bytes of a
   64-byte SHAKE256 output are its 32-byte output. FIPS 202 section
   6.2. This is the opposite of rule 1, and of BLAKE2's rule, where the
   digest length is part of the hash.
3. **SHA3-256 and SHAKE256 have the same rate and are different
   functions.** They differ in one byte of padding, `0x06` against
   `0x1f`. A 32-byte SHAKE256 output is not a SHA3-256 digest. FIPS 202
   sections 6.1 and 6.2.
4. **The number in `SHAKE128` is a security strength, not a length.**
   SHAKE128 gives 128 bits of security at every output length. That is
   why this package has a `SHAKE128_SECURITY_BITS` and no
   `SHAKE128_DIGEST_BYTES`.
5. **A fixed-width hash refuses an output buffer of the wrong length.**
   `finish` and `digest_into` answer `Sha3OutputBufferWrong` with both
   numbers rather than truncating. A caller who wants 20 bytes of
   Keccak output wants `sha3xof`.
6. **`update` takes any chunk length.** The sponge keeps its own
   position in the rate, so chunks need not line up with it and a
   caller never buffers. crypto-nv's SHA-2 `update` takes whole blocks;
   this one does not.
7. **cSHAKE with an empty function name and an empty customization
   string is SHAKE.** SP 800-185 section 3.3 requires it, padding byte
   and all. An implementation that used `0x04` with an empty prefix
   would compute a function nothing else computes.
8. **A KMAC's output length is part of the message it authenticates.**
   SP 800-185 section 4.3 appends `right_encode(L)` before the final
   absorb, so a 32-byte KMAC and a 64-byte KMAC of the same message
   under the same key are unrelated values. `kmac_finish` learns the
   length from the buffer it is handed.
9. **KMACXOF is a different function from KMAC.** It appends
   `right_encode(0)` instead, which is what makes its output
   extendable. A protocol must say which it uses.
10. **A zero-length KMAC is refused**, as `Sha3EmptyOutput`. The
    extendable-output form accepts an empty buffer, because there the
    length is not part of the message.
11. **`left_encode(0)` is the two bytes `01 00`.** SP 800-185 section
    2.3.1. The length written is a count of bytes, and the one written
    by `encode_string` is a count of **bits**. Those two are where a
    hand-written KMAC goes wrong.
12. **Absorbing after squeezing has begun is refused**, as
    `Sha3AbsorbAfterSqueeze`. A sponge pads once.
13. **KMAC is not HMAC.** HMAC applies the key twice because SHA-2
    leaks its state at the end of a message. A sponge does not leak its
    state, so KMAC absorbs the key once and costs one pass.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers `sha3keccak` and `sha3sponge`. They
contain only integer arithmetic over fixed-size values.

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds today:

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

The probe produces a Cortex-M4 executable that runs the permutation,
absorbs eight bytes into a sponge a byte at a time, seals it and reads
thirty-two bytes of output back. It builds and it is not run: every
function it calls is a `todo()` today.

The consumer for this is a bootloader. It has an image in flash, a
digest it was provisioned with, and no heap. The 200-byte state lives
in its own stack frame, with no header, no reference count and nothing
to free.

**A device cannot use the other four modules.** They speak `Bytes`,
`Str` and `Result`, and the embedded runtime defines none of them. One
host-only function anywhere in a compilation unit is an undefined
symbol at link time on a device, whether or not the firmware calls it.

One detail of the low-level interface matters for correctness. A byte
fed to a sealed sponge is ignored, and a byte squeezed from an unsealed
one is zero. Neither is an error, because a function over a plain value
has nowhere to put one. `sha3sponge.is_sealed` is the question to ask,
and the `Bytes` layer above turns the same mistake into
`Sha3AbsorbAfterSqueeze`.

## Timing behaviour

- **The permutation is constant-time by construction.** Its steps use
  only exclusive-or, and, not, and rotation by fixed amounts. No branch
  depends on a message or key bit, and no table is indexed by secret
  data.
- **Absorbing and squeezing branch on how many bytes are buffered**,
  that is, on the message length and the output length. Both are public
  in every construction this package is for.
- **The state accessors `lane`, `state_byte` and `squeeze_byte` are
  indexed by an argument the caller chooses.** They are constant-time
  with respect to the key, not with respect to that index. Nothing in
  this package passes a secret as an index.
- **This package does not compare tags.** Checking a KMAC needs a
  constant-time comparison, and the one on the registry is
  `digest.ct_eq` in
  [crypto-nv](https://novo-lang.org/packages/crypto-nv). A program that
  verifies a tag calls that function itself. This package does not
  depend on crypto-nv, so that a device which only needs SHA-3 does not
  have to link SHA-2, SHA-1, MD5 and HMAC as well.

## What is not included

- **TupleHash and ParallelHash.** SP 800-185 defines two more
  functions on cSHAKE. TupleHash hashes a sequence of strings
  unambiguously; ParallelHash is a fixed tree. Both are worth having
  and neither is needed by anything on the registry yet.
- **RawSHAKE, and Keccak as it was submitted.** The original Keccak
  used different padding from the standardised SHA-3, so its digests
  differ from every function here. Software that predates FIPS 202 —
  Ethereum's `keccak256` among it — computes the original. This package
  implements the standard.
- **SHA3-512/256 or any other truncation.** FIPS 202 defines no
  truncated variants, unlike SHA-2, which has SHA-512/256.
- **A constant-time comparison.** See "Timing behaviour".
- **An HMAC wrapper.** KMAC is the keyed construction for Keccak, and
  it costs one pass. See rule 13.
- **Any input or output.** A message arrives as bytes the host read,
  and a digest is written into a buffer the caller owns.

## Related packages

- [crypto-nv](https://novo-lang.org/packages/crypto-nv) is SHA-256,
  SHA-512, SHA-1, MD5 and HMAC. Take it when a protocol names SHA-2.
  Take this package when a protocol names SHA-3, SHAKE or KMAC. The two
  families are not substitutable in either direction.
- [blake2-nv](https://novo-lang.org/packages/blake2-nv) is BLAKE2b and
  BLAKE2s, which were the other finalist family. BLAKE2 is keyed
  directly, like KMAC, and is faster in software. SHA-3 is what
  standards specify.
- [asn1-nv](https://novo-lang.org/packages/asn1-nv) names the SHA-3
  object identifiers a certificate's signature algorithm is written
  with.
- [base64-nv](https://novo-lang.org/packages/base64-nv) and
  `bytes.to_hex` are how a digest is printed. This package answers
  bytes.

## Test vectors

FIPS 202 is the reference for the hashes and the XOFs, and NIST's
Cryptographic Algorithm Validation Program publishes the byte-oriented
known answers as `SHA3_256ShortMsg.rsp` and its siblings. SP 800-185's
example values supply the cSHAKE and KMAC vectors. The implementations
to check a port against are RustCrypto's `sha3` crate and CPython's
`hashlib`.

```bash
novo test tests/sha3_vectors_tests.nv   # the known answers
novo test tests/sha3_sponge_tests.nv    # the permutation and the sponge
novo test tests/sha3_kmac_tests.nv      # the SP 800-185 encodings, cSHAKE, KMAC
novo test tests/sha3_cover_tests.nv     # every public function is reached
```

The suite carries the digests of the empty message and of `"abc"` at
all four widths, SHAKE128 and SHAKE256 of the empty message at 32
bytes, cSHAKE128 sample 1, KMAC128 samples 1 and 2, the first and last
round constants, the rates and capacities of all eight functions, and
each of the three SP 800-185 length encodings at its awkward values. It
asserts that a narrow digest is not the front of a wide one, that a
short SHAKE output is the front of a long one, that SHAKE256 is not
SHA3-256, that cSHAKE with two empty strings is SHAKE, that a KMAC at
two lengths gives unrelated tags, and that a byte absorbed after
sealing is ignored.

The tests compile today and fail at run, each on the `not implemented:
sha3-nv.<module>.<fn>` panic that is its body. That is the expected
state of an interface release. They turn green one at a time as bodies
land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The four `SHA3_*` geometry constants, the eight width constants, the four SHAKE constants, the three pad bytes, `KMAC_FUNCTION_NAME` | yes (they are constants) |
| `sha3keccak.Sha3Lanes`, `sha3sponge.Sha3Sponge`, `sha3hash.Sha3State`, `.Sha3Width`, `sha3xof.Sha3Xof`, `.Sha3XofKind`, `sha3kmac.Sha3Kmac`, `sha3err.Sha3Error` | the types are declared |
| `sha3keccak.zero_lanes`, `.lane`, `.with_lane`, `.state_byte`, `.xor_byte` | no |
| `sha3keccak.rotl64`, `.round_constant`, `.rho_offset`, `.pi_index` | no |
| `sha3keccak.theta`, `.rho_pi`, `.chi`, `.iota`, `.keccak_round`, `.permute` | no |
| `sha3sponge.sponge`, `.absorb_byte`, `.seal`, `.squeeze_byte`, `.squeeze_next` | no |
| `sha3sponge.rate_of`, `.capacity_of`, `.position`, `.pad_of`, `.is_sealed`, `.lanes_of` | no |
| `sha3hash.digest_bytes`, `.rate_bytes`, `.capacity_bits`, `.width_name`, `.width_named` | no |
| `sha3hash.new`, `.update`, `.finish`, `.digest_len_of`, `.is_finished`, `.digest_into` | no |
| `sha3hash.sha3_224`, `.sha3_256`, `.sha3_384`, `.sha3_512` | no |
| `sha3xof.xof_rate_bytes`, `.xof_security_bits`, `.xof_name` | no |
| `sha3xof.xof_new`, `.xof_update`, `.xof_seal`, `.xof_squeeze`, `.xof_is_sealed`, `.xof_squeezed` | no |
| `sha3xof.shake128_into`, `.shake256_into`, `.shake128`, `.shake256` | no |
| `sha3kmac.left_encode`, `.right_encode`, `.encode_string`, `.bytepad` | no |
| `sha3kmac.cshake_new`, `.cshake128_into`, `.cshake256_into` | no |
| `sha3kmac.kmac_new`, `.kmac_update`, `.kmac_finish`, `.kmac_finish_xof`, `.kmac_rate_bytes` | no |
| `sha3kmac.kmac128`, `.kmac256`, `.kmac128_xof`, `.kmac256_xof` | no |
| `sha3err.is_length_fault`, `.is_order_fault`, `.code`, `Sha3Error.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
