# ClipRelay

End-to-end encrypted clipboard-text and file sync between devices that share a room code. Clients
connect out over WebSocket to a relay that forwards opaque XChaCha20-Poly1305 blobs and cannot
decrypt them. A Rust 2024 workspace (stable toolchain): `cliprelay-core` (protocol, crypto, limits),
`cliprelay-relay` (axum server, Linux or Windows) and `cliprelay-client` (egui tray app, Windows
only; `ClipRelay.exe`). Public repository `Swatto86/ClipRelay`, single branch `master`, MIT, latest
release v1.0.1.

## Build, test, verify

- Run: `cargo run -p cliprelay-relay -- --bind-address 0.0.0.0:8080` (endpoints `/ws`, `/healthz`);
  `cargo run -p cliprelay-client -- --server-url <ws-url> --room-code <code> --client-name <name>`
  (default server `wss://relay.swatto.co.uk/ws`). Two clients in one room test a transfer.
- No verify script. CI (`ci.yml`) runs on Ubuntu: `cargo fmt --all -- --check`,
  `cargo clippy -p cliprelay-core -p cliprelay-relay -- -D warnings`, `cargo check`,
  `cargo test -p cliprelay-core`, `cargo test -p cliprelay-relay --test e2e_relay`; on Windows:
  `cargo clippy -p cliprelay-client -- -D warnings`, `cargo test -p cliprelay-client`. The client
  compiles to a stub off Windows, so its lint and tests only mean something on Windows.
- Release: push a `v*.*.*` tag. `release.yml` tests and builds the Linux `cliprelay-relay`,
  `ClipRelay.exe` and an NSIS `ClipRelay_<ver>_x64-setup.exe` (`.github/installer.nsi`), then
  publishes all three. The version lives only in the root `Cargo.toml` `[workspace.package]`.
- `update-application.ps1 [-Version x.y.z] [-Notes ...] [-DryRun] [-Force]` bumps that version,
  runs build, fmt, clippy and the full test suite, commits, tags, pushes, then deletes every older
  `v*.*.*` tag and GitHub release. Use `-DryRun` unless the owner asks for a release.
- Relay deployment: `sudo deploy/install-relay-systemd.sh --binary <path> [--bind-address ...]`
  installs `/opt/cliprelay/bin/cliprelay-relay`, `/etc/cliprelay/relay.env` and the hardened
  `cliprelay-relay.service` (user `cliprelay`), behind Caddy for TLS.

## Where things are

- `cliprelay-core/src/lib.rs`: framing, room key derivation, encryption, replay checks, limits.
- `cliprelay-relay/src/lib.rs` (router, rooms, rate limiting), `main.rs` (CLI),
  `tests/e2e_relay.rs`.
- `cliprelay-client/src/main.rs`: the whole Windows app (UI, tray, hotkey, runtime, transfers);
  `lib.rs`: autostart registry helpers; `ui_state.rs`, `ui_layout.rs`; `tests/`.
- `.deploy-relay-src/` is a stale 0.1.0 copy of core and relay that nothing builds or references.
  Change the live crates, not it.

## Constraints

- The relay never decrypts, stores or persists anything; it holds only in-memory room membership.
  Offline clients miss messages.
- Limits: clipboard text 256 KiB, relay frame 300 KiB, 10 devices per room, files up to 200 MiB
  in 64 KiB chunks (at most 4096), 8 in-flight receives, 10 min transfer timeout. The relay
  rate-limits each connection (burst 400, 200/s) and the client paces chunks to fit.
- Room key: HKDF-SHA256 over SHA-256 of the room code, salted with a hash of the sorted,
  length-prefixed device IDs. Replay counters are monotonic per sender. Room code, server URL and
  device name are trimmed before saving so peers derive the same key.
- The client sends a WebSocket ping every 30 s so reverse proxies keep idle connections open.
- eframe does not run `update()` while the window is hidden, so tray and hotkey callbacks show
  the window with Win32 calls directly; keep that path (see `PROJECT_ATLAS.md`).
- Client data: `%LOCALAPPDATA%\ClipRelay\` holds `config.json`, `ui_state.json` and
  `cliprelay-client.log`; received files go to `Downloads\ClipRelay`. `CLIPRELAY_CONFIG_DIR` and
  `CLIPRELAY_DATA_DIR` override these; `CLIPRELAY_DEBUG=1` opens a console for `RUST_LOG` output.
  "Start with Windows" writes an HKCU Run entry with `--background`.

## Open questions

- `ci.yml` triggers on pushes to `main`, but the branch is `master`: pushes run no CI (pull
  requests and manual runs still do).
- `README.md` is stale in places (50 MiB file cap, tray colour meanings, the room-key salt);
  `PROJECT_ATLAS.md` and the code agree with each other.
- The NSIS uninstaller leaves the autostart Run entry in place.

## Where longer material lives

- `README.md`: overview, relay install on Linux with Caddy, tray client guide.
- `PROJECT_ATLAS.md`: the system map: boundaries, patterns, diagnostics, limits, invariants.
- `docs/HOW_IT_WORKS.md`: protocol, security model, UX, cloud deployment, troubleshooting.
- `FEATURE_SUMMARY.md`: older feature notes.
