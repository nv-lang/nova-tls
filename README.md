# nova-tls

TLS client/server for [Nova](https://nv-lang.org) — client + server handshake,
certificate-verification modes (system roots / custom CA / SPKI pinning /
insecure-skip-verify for tests), mutual TLS (mTLS), and `io.Read`/`io.Write`
conformance on top of `std.net.TcpStream`.

Backend: [mbedTLS](https://www.trustedfirmware.org/projects/mbed-tls/) via a
thin C shim (`native/tls_c_shim.c`) — pure C, no Rust/cargo required to build
this package. mbedTLS itself ships as a prebuilt static library (vcpkg or a
system package); this package never builds mbedTLS from source.

Extracted from the Nova monorepo's `std/tls` (Plan 116 core design + Plan 195
mbedTLS backend swap) into a standalone repository per
[Plan 193](https://github.com/nv-lang/nova/blob/main/docs/plans/193-nova-tls-repo.md)
— the reference instance of Nova's native-module pattern
([Plan 195](https://github.com/nv-lang/nova/blob/main/docs/plans/195-native-modules-c-not-rust.md):
`.nv` facade + `.c` shim + prebuilt `.lib`, wired through `[ffi]`, zero Rust).
Public API is unchanged from `std.tls` — only the module path moved
(`std.tls.*` -> `tls.tls.*`; see "Module path" note below).

## Usage

```nova
import tls.tls.{TlsStream, ClientConfig, VerificationMode}
import std.net.{Net, TcpStream, SocketAddr, real_net}

fn fetch(host str, port u16) Net -> Result[(), TlsError] {
    ro tcp = TcpStream.connect(SocketAddr.new(host, port))!
    ro cfg = ClientConfig.new(host)
    consume stream = TlsStream.connect(tcp, cfg)!
    stream.write_all("GET / HTTP/1.1\r\nHost: ${host}\r\n\r\n".bytes())!
    ro resp = stream.read_to_vec(4096)!
    stream.close()
    Ok(())
}
```

## Layout

```
nova-tls/
├── nova.toml            [package] name = "tls"; [lib] src = "src"; [ffi] native shim
├── native/
│   ├── tls_c_shim.c      mbedTLS backend (compiled via [ffi] c_shims)
│   ├── tls_shim.h         C-side prototypes (Nova <-> C ABI contract)
│   └── tls_mozilla_roots.h  compiled-in Mozilla root CA bundle
└── src/tls/
    ├── ffi.nv             extern "C" fn declarations against native/tls_c_shim.c
    ├── error.nv            TlsError (typed error surface)
    ├── config.nv           ClientConfig / ServerConfig / VerificationMode / ClientCertMode
    ├── client.nv           TlsStream.connect (client handshake)
    ├── server.nv           TlsStream.accept (server handshake)
    ├── stream.nv           TlsStream (sans-I/O pump + io.Read/io.Write conformance)
    ├── *_test.nv           peer tests (same-module, positive)
    ├── neg/                EXPECT_COMPILE_ERROR fixtures (standalone CUs)
    └── testdata/           self-signed fixture certs (server + mTLS client CA/leaf)
```

## Module path

D78 (`module = parent_dir.target`, spec/decisions/07-modules.md) always
prefixes a root-level file or folder-module with the **package name**
(`[package] name = "tls"`) as `parent`. Since this package's whole surface
lives in one folder-peer domain also named `tls/` (`src/tls/*.nv`), the
resulting module is `tls.tls` (not the bare `tls` shown as the aspirational
example in the D78 external-package-naming amendment, `spec/decisions/
07-modules.md` "Именование внешних пакетов-репозиториев" / `docs/guide/
authoring-a-module.md` §8) — confirmed empirically against the compiler's
`E_D78_MODULE_PATH_MISMATCH` check (compiler-codegen/src/manifest.rs
`check_module_path_with_kind`). A bare single-segment `tls` module is not
reachable for a root-level item under the current rev-3 implementation
(tested: neither a lone `src/tls.nv` facade nor flattening the domain files
directly into `src/*.nv` produces it — flattening additionally breaks the
same-module peer relationship the facade files rely on). Import as
`import tls.tls.{TlsStream, ...}`.

## Building standalone

Requires the Nova toolchain (`nova` CLI + clang) and a prebuilt mbedTLS
(`mbedtls`/`mbedx509`/`mbedcrypto` static libs + headers — e.g.
`vcpkg install mbedtls`). No Rust/cargo.

```sh
# mbedTLS include/lib dirs must be reachable by the linker (system search
# path, or prepend to LIB/LIBRARY_PATH before invoking nova).
#
# Boehm GC (mandatory Nova runtime dep) needs its own lib/include dirs —
# point NOVA_GC_LIB_DIR (+ optional NOVA_GC_INCLUDE_DIR) at a prebuilt
# bdwgc if it isn't reachable via the default vcpkg/system lookup
# (see compiler-codegen/src/test_runner.rs detect_boehm).
#
# `nova` does not (yet) bundle/locate the standard library relative to the
# nova.exe install — a standalone package must point it at a Nova checkout's
# std/ via NOVA_STD_PATH (compiler-codegen/src/manifest.rs resolve_std_path):
export NOVA_STD_PATH=/path/to/nova/std

# Use `nova test`, not `nova build <single-file>`, for anything beyond a
# syntax/import smoke check — this package has no `main`, and isolated
# single-file builds of a library CU can hit generic-inference ambiguities
# that a full test CU resolves via its own call sites (verified upstream:
# the same `ffi.nv` hits it identically inside the Nova monorepo).
nova test src/tls
```

**Known blocker (Plan 193 Ф.1, as of 2026-07-11):** `nova test` currently
requires a full copy of the Nova compiler's C runtime — `compiler-codegen/nova_rt/`
(~64 files) plus the `libuv` submodule (~468 MB source, built on demand) — to
exist *inside this package's own repo root* (`nova-tls/compiler-codegen/nova_rt/`).
The lookup (`RepoPaths::rt_dir`/`cg_include` in `nova-cli/src/main.rs`,
`detect_or_build_libuv` in `compiler-codegen/src/test_runner.rs`) is hardcoded
relative to the package root, with no env-override (unlike `NOVA_STD_PATH` /
`NOVA_GC_LIB_DIR` above). Until the toolchain gains a way to resolve its own
runtime relative to the `nova.exe` install (or an env override symmetric with
`NOVA_STD_PATH`), a genuinely standalone `nova test` run isn't possible without
vendoring the compiler's runtime into this repo — see the Nova monorepo's
`docs/plans/193-nova-tls-repo.md` "Ф.1 блокер" section for the full trace.

## License

Dual-licensed under [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at
your option — same terms as the Nova compiler and standard library.
