# Changelog — SimForge S4

## 0.2.0 — Phase 2

### Added
- **Ed25519 signing infrastructure.** Vendored TweetNaCl (public domain) in
  `third_party/tweetnacl/`; new `src/core/ed25519` wrapper (keypair, sign,
  verify, base64, key-file I/O). Licenses and update manifests are now signed
  with Ed25519; `verify.pub` (32-byte public key, base64) ships with the app.
- **`sfs4_sign` CLI** (`tools/sign`): `keygen`, `sign-license`,
  `sign-manifest`, `verify`. Replaces the Phase-1 PowerShell signing scripts
  (removed: `gen_keys.ps1`, `sign.ps1`, `sign_license.ps1`,
  `sign_manifest.ps1`, `common_canonical.ps1`).
- **OGFNIL companion** (`src/companion/ogfnil.*` → OGFNIL tab): observation-only
  launcher/game integration — auto-detection of the launcher and game
  executables (config override, then standard locations), Toolhelp32 process
  scan with `PROCESS_QUERY_LIMITED_INFORMATION` path confirmation, game
  version from the exe file's version resource, `ShellExecuteW` launcher
  start, and a 2s lifecycle monitor (start/exit/restart/unexpected-exit
  events, logged + surfaced in the UI). No injection, no memory access, no
  window sync, no input — the synthetic sims remain fully in-process.
- **Config profiles** (`src/common/profiles.*`): named profiles under
  `%LocalAppData%\SimForgeS4\profiles\`, save/load/delete/apply from the
  OGFNIL tab; `apply_sim` copies only the simulation subset so server and
  companion fields survive a profile load. Optional auto-apply on game start.
- **Licensing server** (`server/main.cpp` → `sfs4_server.exe`): minimal
  HTTP/1.1 server on Winsock (loopback-only bind, no URL ACL/elevation),
  JSON file DB with atomic writes. Endpoints: `/v1/health`,
  `/v1/activate` (device registration + seat limits), `/v1/heartbeat`
  (server re-verifies signature + revocation), `/v1/deactivate`,
  `/v1/admin/issue`, `/v1/admin/revoke` (loopback + `X-Admin-Token`).
  Signing keys auto-generate on first run. `scripts/run_server.ps1` added.
  All request handling is exception-guarded — malformed JSON or wrong-typed
  fields return HTTP 400 and never terminate the process.
- **Client activation + heartbeat.** `lic::activate_remote` posts to the
  server, verifies the returned signature locally before accepting;
  background heartbeat thread (5 min interval) with a 5-minute grace window
  on revocation, then graceful exit. Offline failures are tolerated.
- **Secure update flow.** Signed manifest (`version|url|sha256` + Ed25519
  sig) → HTTPS download → SHA-256 verification → staged package +
  `apply-update.cmd` with atomic swap and automatic rollback (including a
  post-start process liveness check). Applied at app exit via
  `upd::apply_staged`.
- **CV Lab research module** (`src/sim/cvlab`): procedural synthetic frame
  generation (deterministic, in-process), detector scoring vs ground truth
  with IoU. Pure-C++ connected-components baseline always available; OpenCV
  threshold+contours path enabled with `-DSFS4_WITH_OPENCV=ON` (find_package,
  nothing fetched). Settings tab + bottom-right detection inset overlay.
- **Automated test suite** (`tests/main.cpp`, CTest): Ed25519
  sign/verify/tamper, base64 roundtrip, SHA-256 FIPS vector, license/manifest
  canonical strings + tamper rejection, config roundtrip + clamps, target
  field determinism/damage/respawn, world→screen projection sanity, CV lab
  determinism + detection.
- **Header audit target** (`sfs4_header_audit`): one TU per public header so
  every header compiles standalone (catches missing includes and cycles).

### Changed
- Version bumped 0.1.0 → 0.2.0 (CMake project, `version.rc`, package script).
- `CMakeLists.txt`: `LANGUAGES CXX C` (tweetnacl is C), `enable_testing`,
  `SFS4_WITH_OPENCV` option; new targets `sfs4_sign`, `sfs4_server`,
  `sfs4_tests`, `sfs4_tweetnacl` (compiled `/W0`).
- `integrity.hpp` now includes `common/config.hpp` (missing dependency fix).
- README-DEV.md rewritten for the Phase 2 architecture; new CHANGELOG.

### Removed
- `src/core/rsa.*` (CNG RSA verify) — superseded by Ed25519 everywhere.
- PowerShell signing scripts (see above).

### Notes
- Canonical strings (byte-exact invariant across client/server/sign tool):
  license = `product|licensee|device_id|issued|expires|plan`;
  update manifest = `version|url|sha256`.
- Server binds `http://127.0.0.1:8085/` (Winsock, loopback-only — no URL ACL
  or elevation required).
- TLS for production licensing traffic terminates at a reverse proxy in
  front of `sfs4_server`.

## 0.1.0 — Phase 1

- Initial vertical slice: D3D11 + ImGui window/renderer, synthetic target
  field (lurker/runner/planted behaviors, 14-bone skeleton, health/shield),
  Legit + Advanced HUD modes, aim/trigger simulation with accuracy stats,
  frame pacer (165 Hz target), integrity module (Authenticode self-check,
  bundle hash manifest, DPAPI-bound config HMAC), file-token licensing,
  update scaffolding, dependency bootstrapper, Inno Setup installer.
