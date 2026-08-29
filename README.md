<p align="center">
  <img src="assets/fluyt_logo.png" alt="Fluyt" width="200">
</p>

<h1 align="center">Fluyt</h1>

<p align="center">
  <img src="https://img.shields.io/badge/version-v0.2.49-blue" alt="Version">
  <img src="https://img.shields.io/badge/macOS-Apple%20Silicon-lightgrey" alt="macOS">
  <img src="https://img.shields.io/badge/Windows-x86--64-lightgrey" alt="Windows">
  <img src="https://img.shields.io/badge/Linux-x86--64-lightgrey" alt="Linux">
  <img src="https://img.shields.io/badge/built%20with-Go-00ADD8" alt="Go">
  <img src="https://img.shields.io/badge/license-as--is-orange" alt="License">
</p>

Collect logs from syslog, vendor APIs, and cloud platforms. Route them to any
SIEM, data lake, or observability stack. Never lose a message — every
destination has its own on-disk buffer, and delivery resumes exactly where it
left off after an outage, a restart, or an update. One binary, one config file,
native service integration on Windows, Linux, and macOS.

This repository holds the released binaries and the signed release manifest.
Integrations (input and output modules) are published separately at
[fluyt-modules](https://github.com/begley-blu/fluyt-modules).

## Highlights

- **Collect from anywhere.** Syslog (UDP and TCP) is built in. Vendor API
  integrations — security tools, cloud platforms, AI compliance feeds, network
  infrastructure — install as modules with a single command.
- **Deliver to anywhere.** HTTP log ingestion is built in. Modules add syslog
  forwarding, Amazon S3, Datadog, Elasticsearch / OpenSearch, Grafana Loki,
  Microsoft Sentinel, Rapid7, and more.
- **Content-aware routing.** Match on input, detected message format, source IP
  or CIDR, or any combination. Fan out a single event to multiple destinations.
- **Zero-loss delivery.** Every output spools to disk independently. The read
  cursor advances only after the destination acknowledges. A failed endpoint
  backs up its own spool and nothing else.
- **Self-updating with rollback.** Fluyt checks for newer releases, verifies
  the download, swaps the binary, and rolls back automatically if the new
  version does not come up.
- **Cross-platform.** Runs as a Windows service (SCM), a systemd unit, or a
  macOS LaunchDaemon. Same binary, same config, same behavior.
- **Management UI.** A native desktop UI on macOS and Windows for configuring
  inputs, outputs, and routes. Full CLI on every platform.
- **Modular and verified.** Install only the integrations you need. Each module
  is cryptographically signed and verified before every launch.

## Integrations

Syslog reception and HTTP log ingestion are compiled into every Fluyt binary and
work with no modules installed.

Everything else — vendor API inputs and additional output destinations — ships as
a signed module package. Browse the full catalog of available integrations at
[fluyt-modules](https://github.com/begley-blu/fluyt-modules), or from the CLI:

```
fluyt module catalog
```

Modules cover security and endpoint tools, identity providers, cloud
infrastructure, AI platform compliance, vulnerability management, and
SIEM / observability destinations.

## Install

Download the binary for your platform from the newest directory under
[`core/`](core/), rename it to `fluyt` (`fluyt.exe` on Windows), and put it
somewhere permanent — the service is registered at whatever path you install it
to, so a binary run from `~/Downloads` will register a service pointing there.

| Platform | Asset | Management UI | Suggested location |
|---|---|---|---|
| macOS (Apple silicon) | `fluyt-darwin-arm64` | yes | `/Applications/Fluyt.app/Contents/MacOS/fluyt` |
| Windows (x86-64) | `fluyt-windows-amd64.exe` | yes | `C:\Program Files\Fluyt\fluyt.exe` |
| Linux (x86-64) | `fluyt-linux-amd64` | no | `/usr/local/bin/fluyt` |

The macOS asset is the executable from inside the application bundle, not the
bundle itself — that is the file `fluyt update` replaces, so an update leaves the
bundle and the service registration pointing at it untouched. Opening the
management UI on macOS needs the surrounding `Fluyt.app`; the bare executable
runs the service and the CLI.

The Linux asset is headless. Every subcommand works and the shipper itself is
identical, but there is no management UI, so a Linux host is configured by editing
`config.json` — see [Configuring a Linux host](#configuring-a-linux-host).

Then register it as a service:

```
fluyt install
```

That writes a default config if there is none, registers the service, and starts
it. It needs administrator on Windows or root elsewhere.

## Getting started

1. Run `fluyt install` (elevated).
2. On macOS or Windows, launch the binary with no arguments to open the
   management UI. On Linux, edit `/etc/fluyt/config.json` directly.
3. Add an **output** — paste the URL and token for your ingestion endpoint and
   save it.
4. Add a **route** — select an input, leave the match empty for a catch-all, and
   point it at your output.
5. Click **Restart** (or `fluyt restart` on Linux) so the service picks up the
   new config.
6. Point your devices' syslog at this host on port 514 and watch the dashboard.

To collect from a vendor API, install the module first:

```
fluyt module get <id>      # e.g. fluyt module get okta_input
fluyt restart
```

Then add the new input and a route for it.

### Configuring a Linux host

`fluyt install` writes a default `/etc/fluyt/config.json`. Edit it, then apply the
change:

```
sudo fluyt restart
sudo fluyt status
```

`fluyt status` reports per-input and per-output health, which is how you confirm a
change took. To see what a config does before committing to it, run it in the
foreground against a scratch copy:

```
sudo fluyt run --config ./test-config.json
```

### Configuration paths

| Platform | Path |
|---|---|
| Windows | `%ProgramData%\Fluyt\config.json` |
| Linux | `/etc/fluyt/config.json` |
| macOS | `/Library/Application Support/Fluyt/config.json` |

On macOS and Windows the management UI reads and writes this file, so it needs
to run elevated. On Linux, edit it with any text editor.

## CLI reference

| Command | What it does |
|---|---|
| `fluyt` | Open the management UI (macOS / Windows) |
| `fluyt run` | Run the log shipper in the foreground |
| `fluyt install` | Register and start the OS service *(needs elevation)* |
| `fluyt uninstall` | Stop and remove the OS service *(needs elevation)* |
| `fluyt uninstall --purge` | Also delete config and spool data |
| `fluyt start` / `stop` / `restart` | Control the installed service |
| `fluyt status` | Print service state and per-input / per-output health |
| `fluyt module catalog` | List modules available to install |
| `fluyt module get <id>` | Download and install a module *(needs elevation)* |
| `fluyt module list` | Show installed modules |
| `fluyt module remove <id>` | Remove an installed module *(needs elevation)* |
| `fluyt module verify` | Verify the integrity of installed modules |
| `fluyt update` | Replace this binary with the published release *(needs elevation)* |
| `fluyt update --check` | Report what is published without changing anything |
| `fluyt version` | Print the version |

Run `fluyt <command> --help` for a command's flags.

## Updating

Fluyt updates itself. It checks this repository for a newer release, downloads
it, verifies the signature and the SHA-256, stops the service, swaps the binary,
and starts it again — restoring the previous binary if the new one does not come
up.

On macOS and Windows a banner appears in the management UI when an update is
available. On Linux, and anywhere you prefer a terminal:

```
fluyt update --check     # report what is available and stop
fluyt update             # download, verify, and apply it
fluyt update --dry-run   # everything except the swap
```

`fluyt update` needs the same privileges as `fluyt install`, because it replaces
a file the service manager runs. An unelevated run tells you the command to
repeat with elevation rather than failing partway through.

Spooled data survives the restart: the delivery cursor is on disk and is only
advanced once an endpoint has acknowledged, so an update neither drops queued
messages nor re-sends delivered ones.

Older releases stay in this repository. To go back to one, download it and run
`fluyt install` again from the new binary's location.

## Modules

Inputs that poll a vendor API — rather than receiving syslog — and output
destinations beyond HTTP ingestion ship as separately signed module packages.
A new integration does not require a new Fluyt release.

Install from the **Add module** page in the management UI, or from a terminal:

```
fluyt module catalog    # what is published
fluyt module get <id>   # download and install
fluyt module list       # what is installed
fluyt module remove <id>
```

Modules carry their own version numbers and update independently of Fluyt
itself. The UI marks the ones with an update available; `fluyt module catalog`
shows the same thing as a list.

Each module is a separate executable, cryptographically signed and verified
before every launch. A module that fails verification does not run, and Fluyt
keeps shipping without it.

Browse the full catalog: [fluyt-modules](https://github.com/begley-blu/fluyt-modules).

### Verify what you downloaded

Every binary's SHA-256 is in [`core/release.json`](core/release.json), which is
signed. Check yours before you install it:

```
shasum -a 256 fluyt-linux-amd64        # macOS, Linux
certutil -hashfile fluyt-windows-amd64.exe SHA256   # Windows
```

## Releases

Each release is one directory:

```
core/release.json               the signed index every host reads
core/v<version>/                the binaries for one release
  fluyt-darwin-arm64
  fluyt-windows-amd64.exe
  fluyt-linux-amd64
  NOTES.md                      what changed, when there is more to say
```

Versions are `<major>.<minor>.<commits>-<short hash>`. The commit count and hash
come from the source history, so a version names exactly one build.

`release.json` names the newest release only; the per-version directories are
kept so an older build stays installable. A release's directory is never
rewritten — a rebuild gets a new version and a new directory, so a URL that
worked once keeps serving the same bytes.

## License

This software is provided as-is, without warranty of any kind.
