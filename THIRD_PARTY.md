# Third-party components vendored in `nova-tls`

This package ships third-party sources inside its own tree. Vendoring means we
**redistribute** those sources, which carries obligations that linking against a
system package does not — so they are listed here explicitly rather than left to
whoever thinks to look inside `native/`.

## Mbed TLS

- **Component**: the TLS implementation behind `TlsStream` / `ServerConfig`;
  compiled from source by the generic `[ffi] vendor_src_dirs` build-and-cache
  mechanism (see `nova.toml`).
- **Vendored at**: `native/mbedtls/` (`include/` + `library/`), version **v3.6.2**
  — see `native/mbedtls/VENDORED.md` for the exact source archive.
- **License**: dual **Apache-2.0 OR GPL-2.0-or-later**, at the recipient's
  choice. Nova takes it under **Apache-2.0**.
- **Full text**: `native/mbedtls/LICENSE` (kept verbatim in the tree, as
  Apache-2.0 §4 requires when redistributing sources).
- **Source**: https://github.com/Mbed-TLS/mbedtls

Nothing in `nova-tls` modifies Mbed TLS sources; the C shim
(`native/tls_c_shim.c`) only calls its public API.

## Relationship to this package's own licence

`nova-tls` itself is **MIT OR Apache-2.0** (see `LICENSE-MIT` / `LICENSE-APACHE`
and the `license` field in `nova.toml`). Choosing Apache-2.0 for both this
package and the vendored Mbed TLS keeps a single, compatible set of terms for
anyone redistributing a binary built from this repository.
