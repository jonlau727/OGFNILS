# SimForge S4 — internal development notes

**Product:** standalone synthetic season-lab (Fortnite Chapter 2 Season 4
*inspired*). Premium, private, closed-source distribution.

**Scope contract — read before adding anything:**
The app renders and simulates its own world. There is no code anywhere in
this tree that opens, reads, injects into, or sends input to another process,
and none will be added. ESP, aim simulation, and trigger simulation are
visualizations of sim state only. This is by design and by spec.

---

## Layout

```
CMakeLists.txt            root build (C++20, /GL+/LTCG, static CRT, no PDB in Release)
cmake/deps.cmake          pinned FetchContent: glm, spdlog, nlohmann/json, imgui (+backends)
src/common/               log, paths, config (JSON), frame pacer/stats, WinHTTP net client
src/core/                 integrity, ed25519 (TweetNaCl), license, updater
src/sim/                  target field, camera/w2s math, aim+trigger sim, cvlab (research)
src/render/               D3D11+ImGui renderer, HUD/ESP overlay, settings UI, CV inset
src/app/                  main window + loop + startup security pipeline + workers
src/boot/                 dependency bootstrapper (console)
server/                   sfs4_server — http.sys licensing server (activate/heartbeat/revoke)
tools/sign/               sfs4_sign — Ed25519 keygen/license/manifest signing CLI
tests/                    automated suite (CTest)
installer/                Inno Setup script + EULA template
scripts/                  build, package, run_server
docs/                     config example, license example
third_party/tweetnacl/    vendored TweetNaCl (public domain)
```

## Build (dev machine only — never shipped)

1. VS2022 Build Tools with "Desktop development with C++" (MSVC v143, Win SDK 10).
2. CMake ≥ 3.24 on PATH.
3. `scripts\build_release.bat` → `build\Release\SimForgeS4.exe`,
   `SimForgeS4Boot.exe`, `sfs4_sign.exe`, `sfs4_server.exe`, `sfs4_tests.exe`.
4. Dev run: `set SFS4_DEV=1` then launch — bypasses the license gate for local dev.
5. Tests: `ctest --test-dir build -C Release` or run `build\Release\sfs4_tests.exe`.
6. Optional OpenCV research build:
   `cmake -S . -B build-cv -DSFS4_WITH_OPENCV=ON` (needs a local OpenCV install;
   `find_package` — nothing is fetched).

Release hardening applied by CMake: `/W4 /permissive-`, `/GL` + `/LTCG`,
`/DEBUG:NONE` (no PDB artifacts), `/OPT:REF /OPT:ICF`, static CRT.

## Crypto (Phase 2)

Ed25519 everywhere (vendored TweetNaCl, public domain). One canonical-string
invariant, byte-exact across client, server, and sign tool:

```
license:  product|licensee|device_id|issued|expires|plan
update:   version|url|sha256
```

Keys (generate with `sfs4_sign keygen <pub> <sec>`):

* `verify.pub` — single-line base64 of the 32-byte public key. Ships in the
  app bundle; `package.ps1` copies it.
* private key — base64 of the 64-byte secret (seed||pub). **Never ships,
  never commits.** Lives on the signing workstation / licensing server.

License tokens: `license.sfs4.json` (fields + `sig`), verified against
`verify.pub` locally — the client never trusts server responses without
re-verifying the signature. Device binding: `device_id` = first 32 hex chars
of SHA-256(`sfs4-device|<MachineGuid>`); `*` = site license.

## Licensing server

`sfs4_server` (minimal HTTP/1.1 server on Winsock — no web-framework or
http.sys deps; binds loopback only, so no URL ACL or elevation needed):

| Endpoint | Auth | Purpose |
|---|---|---|
| `GET /v1/health` | — | liveness |
| `POST /v1/activate` | license key | registers device (seat-limited), returns signed license |
| `POST /v1/heartbeat` | signed license | server re-verifies sig + revocation; 200/403/404 |
| `POST /v1/deactivate` | license key | frees a seat |
| `POST /v1/admin/issue` | loopback + `X-Admin-Token` | issue a key (`licensee`, `plan`, `seats`, `days`) |
| `POST /v1/admin/revoke` | loopback + `X-Admin-Token` | revoke a key |

Storage: `server/data/licenses.json`, atomic writes (temp + rename). Signing
keys auto-generate on first run into `server/data/`. Admin endpoints require
both the header and a loopback peer address. TLS in production terminates at
a reverse proxy in front of the server.

Run: `scripts\run_server.ps1` (or reserve the URL ACL once:
`netsh http add urlacl url=http://127.0.0.1:8085/ user=Everyone`).

Client heartbeat: every 5 minutes, full license + sig posted. Revocation gets
a 5-minute grace window, then the app exits gracefully. Network failures are
tolerated (offline play stays inside the license terms).

## Updates

End-to-end Ed25519: manifest fetch → signature verify → HTTPS download →
SHA-256 check → stage to `%LOCALAPPDATA%\SimForgeS4\update\stage.exe` →
`apply-update.cmd` (atomic swap + rollback, launched elevated at app exit).
The install dir is untouched until the swap runs; a failed swap restores
`SimForgeS4.exe.old` automatically.

## Distribution flow

```
build_release.bat → signtool (cert thumbprint) → package.ps1
                  → iscc installer\simforge.iss → sign the Setup exe too
```

* `dist/app/` gets `SimForgeS4.exe`, `SimForgeS4Boot.exe`, `config.json`,
  `verify.pub`, `bundle.sha256.json` (hash manifest consumed at runtime).
* Never copy private keys, `.pdb`, or source archives into `dist/`.

## Security model summary

| Layer | Mechanism | Where |
|---|---|---|
| App integrity | Authenticode self-check (WinVerifyTrust) | core/integrity |
| App integrity | bundle.sha256.json file-hash verification | core/integrity |
| Config integrity | DPAPI-bound HMAC over config.json bytes | core/integrity |
| Licensing | Ed25519-signed license tokens, device-bound, expiry, DPAPI session cache | core/license |
| Licensing | server-side activation, seat counting, revocation, heartbeat | server/ |
| Updates | Ed25519-signed manifests, sha256 packages, staged atomic swap + rollback | core/updater |
| Deps | official MS endpoints, Authenticode + publisher-CN check before execution | boot |

## CV Lab (research, optional OpenCV)

`sim/cvlab` generates fully procedural frames (targets on a noisy synthetic
background — zero external imagery, zero capture) and scores a detector
against ground truth with IoU. Ships with a pure-C++ connected-components
baseline; when built with `-DSFS4_WITH_OPENCV=ON` the detector path uses
OpenCV threshold + contours. Results surface in the settings' CV Lab tab and
the bottom-right detection inset. This is in-process research tooling only.

## Phase roadmap

**Phase 1:** vertical slice — window/renderer, synthetic target field,
Legit + Advanced HUD modes, aim/trigger simulation with stats,
integrity/license/update modules, bootstrapper, installer + signing scripts.

**Phase 2 (this tree):**
- Ed25519 licensing + update signing (TweetNaCl vendored; RSA path removed)
- licensing server: activation, device seats, revocation, heartbeat, admin API
- client heartbeat thread with revocation grace window
- secure updates: signed manifest → HTTPS download → SHA-256 → staged swap + rollback
- header audit target (every public header compiles standalone)
- automated test suite (CTest): crypto vectors, canonical strings, config
  roundtrip/clamps, target determinism/damage/respawn, w2s, cvlab
- CV Lab optional module (`SFS4_WITH_OPENCV`) + overlay inset

**Phase 3:** localization, licensing portal, hardened installer branding,
signed docs, telemetry-free local crash reporting.
