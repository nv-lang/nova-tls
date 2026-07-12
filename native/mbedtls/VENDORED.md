# Vendored mbedTLS

Upstream: https://github.com/Mbed-TLS/mbedtls
Version: v3.6.2 (tag), source archive `Mbed-TLS-mbedtls-v3.6.2.tar.gz`
License: Apache-2.0 OR GPL-2.0-or-later (see `LICENSE`; nova-tls itself uses
the Apache-2.0 branch)

Only `include/` and `library/` are vendored (public headers + all library
translation units, upstream's own default full-feature `mbedtls_config.h`,
unmodified). Dropped: `3rdparty/` (Everest/p256-m — both disabled in the
default config, not referenced by any compiled `.c`), `programs/`, `tests/`,
`doxygen/`, `scripts/`, `visualc/`, `cmake/`, `docs/`, `framework/` — none of
those are inputs to the plain-`cc` build this package uses (Plan 193 Ф.2
gate-3 "195-pattern": vendored C sources -> cc -> cache -> `[ffi] lib_dirs`,
mirrors the monorepo's `detect_or_build_libuv` precedent — no CMake, no
upstream build system).

Build: driven generically by `[ffi] vendor_src_dirs` in `nova.toml` — see
that file's comments and `compiler-codegen/src/test_runner.rs`
(`build_missing_vendor_ffi_libs`) in the main `nova` repo. First `nova test`
(or any build consuming this package's `[ffi]`) compiles every `.c` directly
under `library/` and archives the result into `native/lib/` under each name
declared in `[ffi] libs` (`mbedtls`, `mbedx509`, `mbedcrypto` — identical
combined archives; static-lib member extraction only pulls objects needed to
resolve outstanding externals, so duplicate content across the three
archives is harmless). Cached — rebuilt only if the archives are missing.

To bump the vendored version: download a newer release tarball, replace
`include/` + `library/` + `LICENSE` wholesale, delete any stale built
archives under `native/lib/` (or let the cache staleness be handled
manually — no content-hash invalidation yet, this is a manual bump).
