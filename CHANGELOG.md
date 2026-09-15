# Changelog

All notable changes to libcrypto-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-15

The first release: forty-nine entry points of the OpenSSL libcrypto C
API, one `@ffi` declaration each, and no logic.

### Added

- `libcrypto` — the whole surface, in six groups.
  - Message digests: the context trio, `EVP_DigestInit_ex`,
    `EVP_DigestUpdate`, `EVP_DigestFinal_ex`, the one-shot
    `EVP_Digest`, the four named digests, the lookup by name and the
    three descriptions.
  - Symmetric ciphers: the context trio, the control call, the padding
    switch, the encrypt and decrypt triples, the four named ciphers,
    the lookup by name and the four descriptions.
  - Message authentication and key derivation: `HMAC` and
    `PKCS5_PBKDF2_HMAC`.
  - Random bytes: `RAND_bytes`, `RAND_priv_bytes` and `RAND_status`.
  - The error queue: `ERR_get_error`, `ERR_peek_error`,
    `ERR_error_string_n`, `ERR_reason_error_string` and
    `ERR_clear_error`.
  - The library: `CRYPTO_memcmp`, `OpenSSL_version`,
    `OpenSSL_version_num` and `X509_free`.
- `tests/libcrypto_tests.nv` — twelve tests over the signatures. They
  call the C library, so they need libcrypto installed. The digest
  tests use the published vector for the message `"abc"`.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Named as missing

**Passing a structure by value.** The novo-lang foreign function
interface passes integers, floats and strings. Every libcrypto entry
point that takes or returns a `BIGNUM`, an `ASN1_TIME` or an
`X509_NAME_ENTRY` by value is absent, and so is every one that takes a
C function pointer.

**The public key half.** `EVP_PKEY`, the signing and verification
calls, the key exchange calls and the whole X.509 certificate surface
are left out of the first release. `X509_free` is here on its own,
because `SSL_get1_peer_certificate` in libssl-sys takes a reference the
caller must drop and this is the call that drops it.
