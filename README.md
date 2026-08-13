# monocypher

Cryptography for [sysl](https://github.com/sysl-lang/sysl) — authenticated encryption, key exchange,
signatures and hashing, bound to [Monocypher](https://monocypher.org) 4.0.3.

**Nothing has to be installed to use this.** Monocypher is two C files that include `<stddef.h>` and
`<stdint.h>` and nothing else, and this package carries them: sysl compiles a library's C as part of
the build, so there is no `-l` flag, no `pkg-config`, and no build script anywhere in this
repository. That is also why there is no `@link` in the binding's header — there is no external
library to name.

It is also why `package.hocon` declares **no capabilities at all**. Monocypher opens no file, reads
no clock, calls no allocator and asks the operating system for nothing, so there is nothing to
require and this builds for a freestanding target as readily as for a host.

```
sh/sysl/monocypher/
    monocypher.sysl         the binding
    tests.sysl              the published test vectors
    c/
        c.sysl              Monocypher as C declares it
        monocypher.c        vendored from LoupVaillant/Monocypher 4.0.3
        monocypher.h
        monocypher-ed25519.c    the optional module: SHA-512 and RFC 8032 Ed25519
        monocypher-ed25519.h
package.hocon               who this package is, and what it needs of the machine
```

**Everything that is C lives in `c/`**, module `sh.sysl.monocypher.c`. That is the two-layer shape every
binding in this organisation uses, and it earns its keep here more than most: the `c` layer has to be
*faithful*, and for a cryptographic library a signature that disagrees with the header links perfectly
and then reads past the end of a key. The layer above it has to be *pleasant*, which is a different
question and would otherwise be answered in the same breath.

**A fixed-size array parameter is why the two layers are not the same thing.** C writes
`const uint8_t key[32]`, which is a pointer with a comment — the compiler checks nothing about it. The
`c` layer declares it as the pointer it is; `monocypher.sysl` takes a slice and `require`s its length
against a `const`, which is the only place the 32 is enforced rather than described.

**There is no `c const` block, and that is a finding rather than an omission.** `monocypher.h` defines
no size macros: the numbers live in those array parameters, and the only `#define`s in either header are
Argon2's three variants, which this binding does not bind. So the sizes come from the algorithms' own
specifications — an X25519 key is 32 bytes because Curve25519 is a 255-bit curve — and asking the C
compiler is not available. Where a header *does* define a constant, asking for it is the rule.

The module is **`sh.sysl.monocypher`**, and the directories are that name: a dotted module name
mirrors its path from the library root. The prefix is the reverse-DNS of `sysl.sh`, so that a package
claims a name nobody else will mint rather than the top-level word `monocypher`.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```hocon
dependencies {
  monocypher { git = "github.com/sysl-lang/monocypher", version = "0.3.0" }
}
```

The coordinate is an identity rather than a URL, so it carries no `https://`, and `version` is the
tag `v0.3.0` here. Resolution clones it, selects versions by MVS, and records what arrived in
`sysl.sum`.

Or build it into an artifact and compile against that, which needs no fetching and is what this
repository's own tests do:

```
sysl build-lib . -o /tmp/monocypher.syslib
sysl run yourprogram.sysl --lib /tmp/monocypher.syslib
```

## Example

A complete worked program lives at
[`sysl-lang/monocypher-example`](https://github.com/sysl-lang/monocypher-example) — a key exchange,
a signature and a sealed message in three files, runnable with `sysl run .`. It is also the shortest
answer to what a sysl project with a dependency looks like. The sketch below is the shape of it.

```sysl
import sh.sysl.monocypher.*

// A key and a nonce you brought from somewhere; see "Randomness" below.
val key = your_random_bytes(32)
val nonce = your_random_bytes(24)

val message = "attack at dawn".bytes
val ad = "message 41".bytes

var cipher: []u8 = [0; 14]
var mac: []u8 = [0; 16]

lock(cipher, mac, key, nonce, ad, message)

var back: []u8 = [0; 14]

if unlock(back, mac, key, nonce, ad, cipher) then
    print(f"recovered: ${back}")
else
    print("someone altered it")
```

Every function writes into storage you already have. Nothing here allocates, and nothing returns a
buffer you have to free — the one exception is `hex`, which builds a `string`.

## ⚠ The two signature schemes are not interchangeable

Both are EdDSA over curve25519, both take a 32-byte seed, and both produce a 64-byte signature — so
nothing in the types tells them apart. They hash with different functions, and a signature made by
one is rejected by the other with no diagnostic beyond `false`.

| use | hashes with | interoperates with |
|---|---|---|
| **`ed25519_sign` / `ed25519_check`** | SHA-512 | everything — this is RFC 8032 Ed25519 |
| `eddsa_sign` / `eddsa_check` | BLAKE2b | Monocypher only |

**Use the `ed25519_` pair unless you have a reason not to.** The BLAKE2b variant is smaller, because
a program already hashing with BLAKE2b needs no second hash compiled in — worth it only when you own
both ends of the link.

## Randomness — this package supplies none, deliberately

Every secret is an input. There is no `generate_key()` here, because Monocypher does not have one
either: an entropy source is the platform's business, and a library that guessed at one would be
guessing about the only part that cannot be checked by a test.

On a hosted target, read 32 bytes from `/dev/urandom` through `sysl.fs`. On a freestanding target,
you know what your entropy source is and the library does not. **A key derived from anything
predictable is not a key**, and no test in this repository can tell you that you got it wrong.

## What it binds

| sysl | does |
|---|---|
| `blake2b(hash, message)` | BLAKE2b; `hash.len` chooses the digest size, 1 to 64 |
| `blake2b_keyed(hash, key, message)` | BLAKE2b as a MAC — no HMAC construction needed |
| `sha512(hash, message)` | SHA-512, for talking to things that are not sysl |
| `lock(cipher, mac, key, nonce, ad, plain)` | encrypt and authenticate, XChaCha20-Poly1305 |
| `unlock(plain, mac, key, nonce, ad, cipher) -> bool` | verify and decrypt; `false` means altered |
| `x25519_public_key(public, secret)` | the public key for a secret key |
| `x25519(shared, your_secret, their_public)` | the shared secret — **hash it before use** |
| `ed25519_key_pair(secret, public, seed)` | RFC 8032 key pair; **the seed is wiped** |
| `ed25519_sign(sig, secret, message)` | sign |
| `ed25519_check(sig, public, message) -> bool` | verify |
| `eddsa_key_pair` / `eddsa_sign` / `eddsa_check` | the BLAKE2b variant — see the warning above |
| `equal(a, b) -> bool` | constant-time comparison, at 16, 32 or 64 bytes |
| `wipe(secret)` | zero a buffer, in a way the optimizer may not delete |
| `hex(bytes) -> string` | lowercase hexadecimal; not cryptography, but everything prints |

Sizes are checked by contract, so a 31-byte key stops here rather than reaching C and reading a byte
past the end of your buffer.

### Three rules the binding cannot enforce for you

- **A nonce must never repeat under one key.** Two messages sealed under the same key and nonce leak
  their plaintexts to anyone holding both. 24 bytes is wide enough that a random nonce per message is
  safe, which is why this binds the XChaCha20 variant rather than the 8- or 12-byte ones.
- **An X25519 shared secret is not a key.** It is a curve point, and the low bits of a curve point
  are not uniformly distributed. Hash it — `blake2b` over the buffer — and that is also where a
  transcript of who was talking to whom belongs.
- **Compare secrets with `equal`, never with `==`.** An ordinary comparison stops at the first
  differing byte, and how long it took is a measurement an attacker can make.

## The tests are the published vectors

```
sysl test .
```

27 tests, none of which check against a value this binding produced. The expected strings come from
the specifications: RFC 7693 for BLAKE2b, FIPS 180-4 for SHA-512, RFC 7748 §6.1 for X25519, RFC 8032
§7.1 for Ed25519. A binding that dropped a length or swapped two arguments could not make one of them
pass by coincidence.

The rest cover what no vector can: that `unlock` rejects each of the four things it authenticates
altered one at a time, that an empty message is not an out-of-bounds index, that the seed really is
wiped, that the two signature schemes really are different, and that every size contract stops rather
than reaching C.

**This needs a sysl newer than 0.0.6.** Until recently only `build-lib` compiled a package's `.c`
files, so `sysl test .` on this repository linked against nothing and failed naming every
`crypto_` symbol. `build-lib` works on 0.0.6; the tests need the fix.

### Not bound

The incremental interfaces (`crypto_blake2b_init` / `_update` / `_final`, and the AEAD's
`crypto_aead_init_*` / `_write` / `_read`) are absent, and **the reason written here has expired.** It
said they needed a shim, because the caller has to allocate a `crypto_blake2b_ctx` or a
`crypto_aead_ctx` and only the header knows how large one is. That was true while nothing but C could
read a `sizeof`, and `c const` now can:

```sysl
c const
    BLAKE2B_CTX_SIZE: usize = "sizeof(crypto_blake2b_ctx)"
```

So what these actually need is an `opaque struct` over storage the caller supplies — the same shape
`regex` uses for `regex_t` — and no C of our own at all. Still worth doing when something has to hash a
stream it cannot hold in memory, and cheaper than this note claimed.

**Argon2** is absent too. It is the password-hashing function and it wants a large caller-allocated
work area, which is a question about allocation policy rather than about cryptography; it deserves
its own decision. Also absent: Elligator, the raw ChaCha20 and Poly1305 primitives, and the low-level
EdDSA scalar operations, all of which are for building constructions rather than using them.

## Upstream

Vendored from [LoupVaillant/Monocypher](https://github.com/LoupVaillant/Monocypher) at release 4.0.3,
from the tarball whose SHA-256 is `8cc9bc341a66249016db9bd70e9142d8d0aef9945973744b1ac05dbc55d8ee66`
— the digest GitHub publishes for that asset, checked before the files were copied.

The binding is ISC; Monocypher is dual-licensed BSD-2 / CC-0. See `LICENSE`, which carries both and
which now says which files each set of terms covers — it carried only Monocypher's until the package
took the two-layer shape, so the binding's own sysl had no stated licence at all. Redistribution
requires.

**Vendoring means upstream fixes do not arrive on their own, and for a cryptographic library that is
a sharper cost than for most.** 4.0.3 itself fixed a timing-leak vulnerability in Ed25519 signatures,
and a copy of 4.0.2 sitting in a repository nobody was watching would still have it. Watch upstream
releases and re-vendor; this is the maintenance this package owes and the reason to prefer a small,
slow-moving library for it.
