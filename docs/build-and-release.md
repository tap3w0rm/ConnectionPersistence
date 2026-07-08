# Build And Release

This project is not implemented yet. This file records the expected direction.

## Development

Expected development commands after scaffolding:

```powershell
cargo run
cargo test
cargo build --release
```

## Packaging

Initial release can be a portable folder:

```text
dist\ConnectionPersistence\
  ConnectionPersistence.exe
  README.md
```

Runtime config and logs should be written to `%APPDATA%` and `%LOCALAPPDATA%`, not beside the executable.

## Startup

The app should register per-user startup through the HKCU Run key only after first valid configuration.

## No-Admin Requirement

Release validation must confirm normal user execution for:

- tray,
- config UI,
- RAS VPN connect,
- route lookup,
- adapter stats,
- sound playback,
- logs,
- startup registration.

If any part requires admin rights, it must be documented and treated as a bug or explicit advanced option.

