[English](README.md) | **Русский**

# nova-tls

TLS-клиент/сервер для [Nova](https://nv-lang.org) — рукопожатие клиента и
сервера, режимы верификации сертификата (системные корни / кастомный CA /
SPKI-пиннинг / insecure-skip-verify для тестов), взаимный TLS (mTLS) и
соответствие `io.Read`/`io.Write` поверх `std.net.TcpStream`.

Бэкенд: [mbedTLS](https://www.trustedfirmware.org/projects/mbed-tls/) через
тонкую C-прослойку (`native/tls_c_shim.c`) — чистый C, для сборки этого
пакета Rust/cargo не нужен. mbedTLS v3.6.2 завендорен в `native/mbedtls/`
(план 193 Ф.2 gate-3, 2026-07-12 — см. `native/mbedtls/VENDORED.md`) и
собирает себя сам при первом `nova test`/`nova build` через общий механизм
сборки-с-кэшированием `[ffi] vendor_src_dirs` — ручная установка через
vcpkg/system не нужна (вручную положенная предсобранная библиотека под
`native/lib/` тоже работает как есть, минуя vendor-сборку).

Извлечено из монорепозитория Nova, из `std/tls` (базовый дизайн — план 116 +
план 195, замена бэкенда на mbedTLS), в отдельный репозиторий по
[плану 193](https://github.com/nv-lang/nova/blob/main/docs/plans/193-nova-tls-repo.md)
— эталонный образец паттерна нативных модулей Nova
([план 195](https://github.com/nv-lang/nova/blob/main/docs/plans/195-native-modules-c-not-rust.md):
фасад `.nv` + C-прослойка + предсобранная `.lib`, подключено через `[ffi]`,
без Rust). Публичный API не изменился по сравнению с `std.tls` — сдвинулся
только путь модуля (`std.tls.*` -> `tls.*`; см. заметку «Путь модуля» ниже).

## Использование

```nova
import tls.{TlsStream, ClientConfig, VerificationMode}
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

## Структура

```
nova-tls/
├── nova.toml            [package] name = "tls"; [lib] src = "src"; [ffi] native shim
├── native/
│   ├── tls_c_shim.c      mbedTLS backend (compiled via [ffi] c_shims)
│   ├── tls_shim.h         C-side prototypes (Nova <-> C ABI contract)
│   └── tls_mozilla_roots.h  compiled-in Mozilla root CA bundle
└── src/
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

## Путь модуля

D78 rev-4 (root peers, `spec/decisions/07-modules.md` «Root peers —
`.nv`-файлы прямо в source root») позволяет `.nv`-файлам, лежащим прямо в
исходном корне пакета (`src/`, согласно `[lib] src` выше), объявлять
однокомпонентную форму `module <package_name>` — группу пиров, аналогичную
`lib.rs` в Cargo. Вся поверхность этого пакета лежит прямо в `src/`
(`src/{client,server,stream,...}.nv`, все объявляют `module tls`), поэтому
итоговый модуль — это голое имя пакета, `tls` — без «статтера». Импортировать
как `import tls.{TlsStream, ...}`, как из `[dependencies]`-потребителя
другого пакета, так и из независимого файла того же пакета (например,
`src/neg/*.nv` использует `import tls.{ClientConfig}` для доступа к этим
пирам).

До амендмента rev-4 этот пакет жил по пути `src/tls/*.nv` по старому правилу
rev-3 `parent_dir.target`, которое заставляло доменную папку повторять имя
пакета (`module tls.tls`, импорт `tls.tls.{...}`) — перенесён на root peers
2026-07-13 (план 202 Ф.3).

## Автономная сборка

Нужен тулчейн Nova (CLI `nova` + clang). Rust/cargo не нужен, ручная
установка mbedTLS не нужна — mbedTLS собирает себя сам из завендоренных
исходников в `native/mbedtls/` при первом запуске (план 193 Ф.2 gate-3;
можно положить предсобранные `mbedtls`/`mbedx509`/`mbedcrypto` под
`native/lib/` вместо этого, если хотите пропустить эту разовую сборку,
например `vcpkg install mbedtls`).

```sh
# Boehm GC (mandatory Nova runtime dep) needs its own lib/include dirs —
# point NOVA_GC_LIB_DIR (+ optional NOVA_GC_INCLUDE_DIR) at a prebuilt
# bdwgc if it isn't reachable via the default vcpkg/system lookup
# (see compiler-codegen/src/test_runner.rs detect_boehm).
#
# `nova` does not (yet) bundle/locate the standard library relative to the
# nova.exe install — a standalone package must point it at a Nova checkout's
# std/ via NOVA_STD_PATH (compiler-codegen/src/manifest.rs resolve_std_path):
export NOVA_STD_PATH=/path/to/nova/std

# Ditto for the compiler's own C runtime (compiler-codegen/nova_rt/ + the
# libuv submodule it needs) — NOVA_CG_INCLUDE / NOVA_RT_DIR, symmetric with
# NOVA_STD_PATH above (resolve_paths in nova-cli/src/main.rs; this is what
# closed the Plan 193 Ф.1 blocker previously documented here — no more need
# to vendor the compiler's runtime into this repo):
export NOVA_CG_INCLUDE=/path/to/nova/compiler-codegen
export NOVA_RT_DIR=/path/to/nova/compiler-codegen/nova_rt

# Use `nova test`, not `nova build <single-file>`, for anything beyond a
# syntax/import smoke check — this package has no `main`, and isolated
# single-file builds of a library CU can hit generic-inference ambiguities
# that a full test CU resolves via its own call sites (verified upstream:
# the same `ffi.nv` hits it identically inside the Nova monorepo).
nova test src
```

## Лицензия

Двойная лицензия — [MIT](LICENSE-MIT) или [Apache-2.0](LICENSE-APACHE), на
ваш выбор — те же условия, что у компилятора Nova и стандартной библиотеки.
