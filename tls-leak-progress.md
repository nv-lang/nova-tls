# [M-187-sustained-live-tls-resource-death] — прогресс (worktree nova-tls-leak, branch fix-tls-leak)

Чекпоинт после сетевого обрыва сессии, 2026-07-17.

## Диагностика (по факту, не гипотеза)

1. Native mbedTLS/шим free-парность проверена ПОЛНОЙ двумя независимыми
   методами:
   - alive-session счётчик (`g_dbg_alive_sessions` в `native/tls_c_shim.c`,
     временный, УБРАН из финального фикса) — возвращался к 0 между
     итерациями Nova-уровневого loopback-repro (400-3000 итераций).
   - Standalone-C harness (вне Nova/GC полностью, `clang` напрямую против
     vendored mbedTLS + `mbedtls_platform_set_calloc_free` custom
     allocator hook) — 1500 полных client+server handshake-циклов через
     РЕАЛЬНЫЕ loopback-сокеты (mbedtls_net_*), SystemRoots-размерный CA-бандл
     (150 сертов) против self-signed loopback-серта (verify FAILS ожидаемо,
     но упражняет ПОЛНЫЙ путь parse+verify+free) — outstanding bytes ПЛОСКАЯ
     линия (4467 baseline, peak 504560 за итерацию), 0 роста за 1500 циклов.
   - Отдельно: голый `mbedtls_x509_crt_parse`+`mbedtls_x509_crt_free` бандла
     3000 раз — working set плато на ~7МБ, 0 роста.
   ВЫВОД: утечка НЕ в нативном mbedTLS/шиме.

2. Nova-уровневый (.nv) repro на loopback (свой diag-инструмент,
   `d:/Sources/nv-lang/nova/examples/tls/loop_leak_diag*.nv`, ВРЕМЕННЫЙ,
   не коммитить/удалить) — client+server TLS round-trip в одном процессе,
   3000 итераций:
   - CustomRoots (маленький серт, УСПЕШНЫЙ handshake) — private-память
     10→179МБ (baseline до фикса).
   - SystemRoots-repro (150-сертный бандл против self-signed, verify FAILS)
     — практически ИДЕНТИЧНАЯ скорость роста (10→179МБ) — т.е. размер CA-
     бандла НЕ является фактором (опровергает начальную гипотезу «утечка в
     парсинге большого бандла»).
   - Контроль БЕЗ TLS (тот же fiber/spawn/Channel/TcpListener каркас, голый
     TCP-эхо) — выходит на плато ~22МБ и НЕ растёт. → утечка специфична
     ИМЕННО TLS-пути, не generic concurrency-каркасу.
   - `gc.collect()` (полный синхронный mark-sweep, `GC_gcollect()`) каждые
     50 итераций — `gc.heap_size()` ВСЁ РАВНО растёт монотонно в лок-степе
     с private-памятью (97МБ→139МБ за 900 итераций несмотря на форс-сборки
     каждые 50). → это НЕ «GC не успевает» — Boehm считает объекты live
     (reachable) даже сразу после полной сборки. Похоже на conservative-scan
     false-positive retention через переиспользуемые fiber-стеки (не
     проверено до конца — вне периметра nova-tls, компилятор/рантайм).

## Фикс (в периметре nova-tls, применён)

Корень НЕ во free-парности (нативно доказано чистой) — фикс снижает
GC-churn/экспозицию, не «переписывает GC» (это было бы вне периметра):

1. `native/tls_c_shim.c` + `src/ffi.nv`: новый extern `tls_pending_out_len`
   (чистый компьют, 0 аллокаций) — `flush_out` (stream.nv) теперь спрашивает
   факт вместо слепой аллокации TLS_CHUNK на каждый вызов.
2. `src/stream.nv`+`src/client.nv`+`src/server.nv`: новое поле
   `TlsStream.scratch []u8` — ОДИН переиспользуемый ciphertext-буфer на весь
   жизненный цикл стрима (заведён в connect/accept ДО конструирования,
   продет через `pump_handshake`/`flush_out`/`fill_from_tcp`/`read_step`/
   `write_step`, перенесён в `TlsStream.new`). `fill_from_tcp` теперь читает
   через `Net.read(tcp, scratch)` напрямую (не `TcpStream.read_to_vec`,
   которая ВСЕГДА аллоцирует фреш `[]u8`).
3. `TLS_CHUNK` 16КиБ→4КиБ (доп. мера — эмпирически размер блока
   пропорционален скорости роста: 16КиБ→2КиБ снизил slope ~56КБ/итер→
   ~19КБ/итер на том же repro).

Комбинированный эффект на своём loopback-repro (3000 итераций,
CustomRoots): baseline ~56КБ/итер → после fix (reuse+4КиБ) ~21КБ/итер
(≈2.6× снижение). Рост НЕ устранён полностью (residual, вероятно
GC/fiber-arena слой — эскалация, не мой периметр), но существенно снижена
экспозиция.

## Осталось (с этого места)

1. Убрать ВРЕМЕННУЮ debug-инструментацию:
   - `src/stream.nv`: export `debug_alive_sessions` УЖЕ убран.
   - `src/ffi.nv`: extern `tls_debug_alive_sessions` — убрать.
   - `native/tls_c_shim.c`: `g_dbg_alive_sessions` счётчик + 3 инкремента/
     декремента + `tls_debug_alive_sessions()` — убрать.
2. `nova test src` (env: NOVA_STD_PATH/NOVA_CG_INCLUDE/NOVA_RT_DIR/
   NOVA_GC_LIB_DIR/NOVA_INCLUDE_DIR) — таргетный гейт корректности
   (29/29 было зелёным на старте волны).
3. Официальный репро-гейт (ОДИН прогон, `examples/flagship/aggregator/
   tls-leak-repro.ps1`, ≤120 запросов, квота open-meteo) — ожидание CLEAN
   exit 0.
4. Вернуть `examples/nova.local.toml` (главный репо, НЕ коммитится) —
   `tls = { path = "../../nova-tls" }` (было временно на `nova-tls-leak`
   для этой волны; ВНИМАНИЕ: файл содержал BOM в оригинале — при восстановлении
   через Edit substring-замену BOM сохраняется автоматически, НЕ переписывать
   Write'ом заново).
5. Удалить временные diag-файлы в главном репо (`examples/tls/
   loop_leak_diag*.nv/.exe`, `examples/tls/*_diag*.log`, `examples/tls/
   sysroots_diag*`, `examples/tls/customroots_diag*`, `examples/tls/
   notls_diag*`, `examples/tls/scratchreuse_diag*`, `examples/tls/
   combo_diag*`, `examples/tls/gcforce_diag*`) — НЕ коммитить, эти файлы
   вне nova-tls-leak worktree (в main nova repo), untracked, просто удалить
   с диска.
6. Backlog-маркер: грепнуть `tls-leak`/`weather-live-oom`/`[M-187-
   sustained-live-tls-resource-death]` в docs/simplifications.md/backlog-
   followups.md (главный репо) — закрыть существующий с числами до/после
   (slope, не «утечка устранена», а «снижена + локализована к GC/fiber-arena
   слою, эскалация»).
7. Финальный коммит(ы) в этом worktree (nova-tls-leak, branch fix-tls-leak).
