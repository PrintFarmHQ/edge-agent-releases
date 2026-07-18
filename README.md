# PrintFarmHQ Edge Agent

The PrintFarmHQ Edge Agent is a small local service that connects the 3D printers and slicers on your network to [PrintFarmHQ](https://printfarmhq.io). It runs on a computer on the same LAN as your printers, discovers and talks to them locally, and syncs state with the PrintFarmHQ cloud — the agent always initiates the connection; the cloud never reaches into your network.

**This repository hosts binary releases only.** The agent's source code is currently developed in a private repository; every release here ships the exact artifacts, an installer script, and SHA-256 checksums, and each release's notes reference the source commit it was built from.

## Install

PrintFarmHQ shows a keyless one-line install command in the dashboard (**Connect Printers** tab). The one-time edge API key is displayed separately:

```bash
curl -fsSL https://github.com/PrintFarmHQ/edge-agent-releases/releases/latest/download/install.sh | sh -s -- --control-plane-url 'https://app.printfarmhq.io/api'
```

To install a specific version instead of the latest, add `--version vX.Y.Z`.

The installer asks for the separately displayed key through `/dev/tty`, or directs you to the loopback dashboard. To update later, re-run the same command: it refreshes the binaries and restarts the service while preserving pairing.

## What the installer does

`install.sh` is deliberately transparent. Step by step, it:

1. Detects your CPU architecture (Apple Silicon `arm64` or Intel `amd64`).
2. Downloads the matching `edge-agent` and `edge-helper` binaries from this repository's release.
3. Verifies every download against the release's `checksums.txt` (SHA-256) and aborts on any mismatch.
4. If it finds a legacy root-managed installation, asks once for administrator access, stops only the recognized owner-specific legacy agent, and transfers the canonical local state to your user without changing its contents or modes.
5. Installs the binaries under `~/.printfarmhq/bin/`.
6. Installs a user LaunchAgent (`io.printfarmhq.edge-agent`) so the agent starts with your session.
7. Installs a root LaunchDaemon (`io.printfarmhq.edge-helper`) with a single `sudo` prompt. The helper is the only privileged process, and it exists solely for operations that require root on macOS (such as standard printer ports and local network aliases used for slicer connectivity).
8. Pairs through the agent's local setup API. Your edge API key is handed over locally and is **never placed in the managed process arguments, logs, or launchd plist files**.

After installation, the agent's local dashboard is available at `http://localhost:18090`.

Flags: `--no-privileged` skips the root helper entirely (some printer features degrade gracefully); `--version` pins a release; `--uninstall` and `--purge` are described below.

## Verify before you run

Piping curl to `sh` requires trust. You don't have to extend it blindly:

```bash
curl -fsSLO https://github.com/PrintFarmHQ/edge-agent-releases/releases/latest/download/install.sh
curl -fsSLO https://github.com/PrintFarmHQ/edge-agent-releases/releases/latest/download/checksums.txt
shasum -a 256 -c checksums.txt --ignore-missing   # verifies install.sh
less install.sh                                    # read what it does
sh install.sh --control-plane-url 'https://app.printfarmhq.io/api'
```

## Supported platforms

- macOS on Apple Silicon (`arm64`) and Intel (`amd64`).
- Linux and Windows are not supported yet.

## Code signing status

Release binaries are currently **unsigned and not notarized**; signing and notarization are planned. The curl-based install does not apply macOS's browser quarantine attribute. If you download an artifact with a browser instead, clear the quarantine flag before running it:

```bash
xattr -d com.apple.quarantine ./edge-agent-darwin-arm64 ./edge-helper-darwin-arm64
```

## Uninstall

- `install.sh --uninstall` removes the services and binaries but preserves local state, so a later reinstall picks up where you left off.
- `install.sh --purge` additionally deletes all local state under `~/.printfarmhq`, including the local virtual-printer certificate authority — after a purge and reinstall, slicers must trust the newly generated CA again.

## Support and security

Questions and problems: email [dev@printfarmhq.io](mailto:dev@printfarmhq.io). Please report suspected security issues to the same address rather than in public.
