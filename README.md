# libcrypto-sys

OpenSSL is a C toolkit for transport security and general-purpose
cryptography. It is two libraries: libssl implements the TLS protocol,
and libcrypto holds the primitives that protocol and everything else is
built from. The library's interface is documented in
[the OpenSSL manual](https://docs.openssl.org/3.0/man3/). This package
declares forty-nine of libcrypto's entry points to novo-lang, one
declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libcrypto. The package contains no logic
of its own, and it does nothing without the C library installed. The
forty-nine entry points cover message digests, symmetric ciphers, keyed
message authentication, key derivation from a password, random bytes
and the error queue; the section "What is not included" says what a
program still cannot do with them alone.

## What it is

A **message digest** is a function that turns any number of bytes into
a fixed number of bytes, such that finding two inputs with the same
output is impractical. SHA-256 produces 32 bytes and SHA-512 produces
64.

OpenSSL reaches every digest through one interface, called EVP. An
algorithm is named by an address: `EVP_sha256` answers the address of a
static description, and that address is what every other call takes.
The same arrangement holds for ciphers.

A **digest context** holds the state of a hash in progress.
`EVP_DigestInit_ex` starts one, `EVP_DigestUpdate` adds bytes any
number of times, and `EVP_DigestFinal_ex` produces the digest.
`EVP_Digest` does all three in one call for a buffer already in memory.

A **symmetric cipher** encrypts and decrypts with one shared key. A
**block cipher** such as AES works on fixed-size blocks, and a **mode**
says how a message longer than one block is handled. Cipher block
chaining (CBC) is the classical mode. Galois/counter mode (GCM) is an
**authenticated** mode: besides the ciphertext it produces a **tag**, a
short value that proves the ciphertext was not altered, and decryption
fails rather than returning altered plaintext.

An **initialisation vector** is a value that makes two encryptions of
the same message under the same key come out different. For CBC it must
be unpredictable. For GCM it is called a **nonce**, and the rule is
stronger: a nonce may never be used twice with one key, and breaking
that rule reveals the key material that authenticates the messages.

A **message authentication code (MAC)** proves that a message came from
someone holding a shared key. HMAC is the construction that builds one
from a digest, and it is specified in RFC 2104.

**Key derivation** turns a password into a key. PBKDF2, specified in
RFC 8018, applies HMAC many times so that guessing passwords costs the
attacker the same many times over.

The **error queue** is per-thread. A libcrypto call that fails pushes
one or more codes onto it and returns a failure indication, and the
caller reads the codes to find out what happened.

## Install

```
novo pkg add libcrypto-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its headers come from the system package `libssl-dev`,
which carries both halves of OpenSSL:

```
sudo apt install libssl-dev
```

On macOS the Homebrew formula is `openssl@3`, and its libraries are not
on the default search path. On other systems OpenSSL builds from its own
source.

## Example

The SHA-256 digest of a message, in one call:

```novo ignore
use libcrypto

fn main() [io, ffi]
    // A buffer is an address and a length; the caller owns both.
    let input = ptr.alloc(32)
    ptr.write_bytes_buf(input, bytes.from_str("abc"))
    // EVP_MAX_MD_SIZE is 64, the length every digest fits in.
    let digest = ptr.alloc(64)
    let length = ptr.alloc_word()

    let ok = libcrypto.evp_digest(input, 3, digest, length,
                                  libcrypto.evp_sha256(), 0)
    if ok != 1
        println("the digest failed, code ${libcrypto.err_get_error()}")
        return
    println("${ptr.read_word(length)} byte digest, first byte ${ptr.load_u8(digest)}")

    ptr.free(length)
    ptr.free(digest)
    ptr.free(input)
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls, with the published digest of
`"abc"` asserted against them, are in `tests/libcrypto_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `libcrypto` | Every entry point, in six groups: message digests, symmetric ciphers, message authentication and key derivation, random bytes, the error queue and the library. |

The six groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Message digests | 15 | Hashes a buffer in one call or a stream in several, and names the four algorithms and their sizes. |
| Symmetric ciphers | 20 | Encrypts and decrypts with four named ciphers, sets the padding and the authenticated-mode parameters, and reports each cipher's lengths. |
| Authentication and derivation | 2 | Computes an HMAC and derives a key from a password. |
| Random bytes | 3 | Fills a buffer from the public or the private generator, and reports whether either is seeded. |
| Error queue | 5 | Reads, describes and clears the errors a failed call left behind. |
| Library | 4 | Compares bytes in constant time, reports the version, and releases a certificate reference. |

## How to choose an entry point

`EVP_Digest` hashes a buffer already in memory. `EVP_DigestInit_ex`,
`EVP_DigestUpdate` and `EVP_DigestFinal_ex` hash a stream the program
reads in pieces. The two produce the same digest.

`EVP_sha256` and its three neighbours name an algorithm at compile
time. `EVP_get_digestbyname` names one at run time, from a string the
program read from a configuration file. The same pair exists for
ciphers.

`RAND_bytes` is for random values the program will publish, such as a
nonce or a salt. `RAND_priv_bytes` is for values it will not, such as a
key. Keeping them apart means an attacker who collects published random
values learns nothing about the private generator.

`ERR_peek_error` leaves the error on the queue and `ERR_get_error`
removes it. Use `ERR_peek_error` when the same error will be read again
further up, and `ERR_get_error` in a loop that drains the queue.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned.
2. **A buffer is an address and a length.** The caller reserves the
   bytes with `ptr.alloc`, writes them with `ptr.write_bytes_buf`, and
   reads them back with `ptr.read_bytes_n` or `ptr.load_u8`.
3. **An entry point that answers a C `int` answers it in 32 bits.**
   Write `as i32` before comparing the answer with a negative number.
4. **A digest buffer must be at least 64 bytes.** That is
   `EVP_MAX_MD_SIZE`, and it is what the reference requires of every
   caller of `EVP_DigestFinal_ex`, whatever digest is in use.
5. **Every `ENGINE *` argument takes 0.** Engines are the superseded
   plugin mechanism. OpenSSL 3.0 answers the default implementation
   for 0.
6. **An output buffer for `EVP_EncryptUpdate` needs room for the input
   length plus the block length minus one.** The call may hold back a
   partial block or emit one held back earlier. OpenSSL manual,
   `EVP_EncryptUpdate`.
7. **`EVP_DecryptFinal_ex` answering 0 means the plaintext is
   worthless.** For an authenticated cipher it means the tag did not
   match, and the bytes already written must be discarded rather than
   used. OpenSSL manual, `EVP_DecryptFinal_ex`.
8. **A nonce may never be used twice with one key.** This applies to
   `EVP_aes_256_gcm` and `EVP_chacha20_poly1305`. Repeating one does
   not merely repeat a ciphertext: it reveals the key that
   authenticates every message under that key.
9. **`RAND_bytes` answering 0 is not a buffer of random bytes.** Check
   the answer. The generator can fail, and the buffer is then whatever
   it was.
10. **Compare a MAC with `CRYPTO_memcmp`.** An ordinary byte-by-byte
    comparison stops at the first difference, and the time it took
    tells an attacker how much of a forged code was right.
11. **The error queue is per-thread and it accumulates.** Call
    `ERR_clear_error` before an operation whose errors are about to be
    read, or an older failure will be reported as this one's cause.
12. **The control commands are numbers.** `EVP_CIPHER_CTX_ctrl` takes
    them, because the C header spells them as macros.

    | Command | Number | Integer argument | Pointer argument |
    | --- | --- | --- | --- |
    | `EVP_CTRL_AEAD_SET_IVLEN` | 9 | the nonce length in bytes | 0 |
    | `EVP_CTRL_AEAD_GET_TAG` | 16 | the tag length in bytes | the buffer to write the tag into |
    | `EVP_CTRL_AEAD_SET_TAG` | 17 | the tag length in bytes | the tag to check against |

## Timing behaviour

`CRYPTO_memcmp` is constant time in the length it is given. It is the
only comparison in this package with that property, and it is the one
to use on a MAC, a tag or any other secret.

The AES implementations OpenSSL selects on a processor with the AES-NI
instructions are constant time. On a processor without them, OpenSSL
falls back to a table-driven implementation whose timing depends on the
key. ChaCha20-Poly1305 is constant time everywhere.

`PKCS5_PBKDF2_HMAC` is deliberately slow, and its cost is the iteration
count multiplied by the cost of one HMAC. The cost is the point: it is
what a password needs to survive being guessed.

The digests are constant time in the length of their input, which is
public in every use here.

## What is not included

- **The public key half.** `EVP_PKEY` and everything built on it — key
  generation, signing, verification, key exchange, RSA, the elliptic
  curves — is left out of the first release.
- **The X.509 certificate surface.** Parsing a certificate, reading its
  subject, its validity dates and its extensions, and building a
  verification chain are all absent. `X509_free` is here on its own,
  because `SSL_get1_peer_certificate` in libssl-sys takes a reference
  the caller must drop.
- **`BIGNUM`.** The arbitrary-precision integers underneath the public
  key algorithms have no entry point here.
- **The BIO abstraction.** `BIO_new`, `BIO_read`, `BIO_write` and their
  neighbours are OpenSSL's own stream abstraction. They are left out of
  the first release.
- **Every entry point that takes a C function pointer.** A novo-lang
  function is not one.
- **Every entry point that passes or returns a structure by value.**
  The novo-lang foreign function interface passes integers, floats and
  strings, and nothing else.
- **The provider and property interface.** `OSSL_PROVIDER_load`,
  `EVP_MD_fetch` and `EVP_CIPHER_fetch` are OpenSSL 3.0's new way of
  selecting an implementation. The older lookup entry points are here
  and answer the default provider's algorithms.
- **The legacy per-algorithm interfaces.** `SHA256_Init`, `AES_encrypt`
  and their neighbours are superseded by EVP and deprecated.

## Related packages

`crypto-nv` is the cryptography written in novo-lang, with no C
library. It is the answer for a program that can use it, and this
package is the escape hatch. It is published.

`libssl-sys` binds the other half of OpenSSL, the TLS protocol. A
program doing TLS takes both, and links one copy of each shared
library.

Choose `crypto-nv` when the program must build for a microcontroller or
for WebAssembly, or when a C toolchain is not wanted. Choose this
package when the program needs an algorithm `crypto-nv` does not carry,
or must produce the same bytes as an existing OpenSSL deployment.

## Test vectors

`tests/libcrypto_tests.nv` holds twelve tests written against the
signatures. They call the C library, so `novo test` needs libcrypto
installed and linkable:

```
novo test tests/libcrypto_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The digest tests assert the published SHA-256 digest of the message
`"abc"`, which is
`ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad`.
The streaming form is fed the same message in two pieces and asserted
against the same bytes. The cipher test encrypts a message with
AES-256-CBC and decrypts it back, and compares the result with
`CRYPTO_memcmp`. The key derivation test derives the same key twice and
compares the two. The rest assert each algorithm's own lengths, that
the random generator is seeded and that the error queue fills and
empties.

## Implementation status

| Group | State |
| --- | --- |
| Message digests | Complete for SHA-1, SHA-256, SHA-512 and MD5, and for anything `EVP_get_digestbyname` finds. |
| Symmetric ciphers | Complete for AES-128-CBC, AES-256-CBC, AES-256-GCM and ChaCha20-Poly1305, and for anything `EVP_get_cipherbyname` finds. |
| Authentication and derivation | Complete for HMAC and PBKDF2. |
| Random bytes | Complete. |
| Error queue | Complete. |
| Library | Complete. |
| Public key | Absent. Left out of the first release. |
| X.509 | Absent apart from `X509_free`. |
| BIO | Absent. Left out of the first release. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

OpenSSL itself is distributed under the Apache 2.0 licence, and
installing it is the reader's own step.
