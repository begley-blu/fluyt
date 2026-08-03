# Fluyt

A universal log shipper. It listens for syslog on the host, sorts messages by
rules you write, and delivers them to an HTTP endpoint — spooling to disk when
the endpoint is unreachable so nothing is lost while it is down.

This repository holds the released binaries and the signed document that
describes them. The source lives elsewhere; there is nothing to build here.

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
it. It needs administrator on Windows or root elsewhere. `fluyt status` reports
service state and per-input and per-output health, `fluyt restart` applies config
changes, and `fluyt uninstall` removes the registration.

On macOS and Windows, run the binary with no arguments to open the management UI,
where you configure inputs, outputs, and routes. `fluyt run` is the same program
in the foreground, which is what the service invokes.

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

### Verify what you downloaded

Every binary's SHA-256 is in [`core/release.json`](core/release.json), which is
signed. Check yours before you install it:

```
shasum -a 256 fluyt-linux-amd64        # macOS, Linux
certutil -hashfile fluyt-windows-amd64.exe SHA256   # Windows
```

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

Inputs that poll a vendor API — rather than receiving syslog — ship as separate
signed packages from a module catalog, so a new integration does not require a
new Fluyt release. Install them from the **Add module** page in the management
UI, or — always, and the only way on Linux — from a terminal:

```
fluyt module catalog    # what is published
fluyt module get <id>   # download one and install it
fluyt module list       # what is installed, and whether it loads
fluyt module remove <id>
```

Modules carry their own version numbers and update independently of Fluyt
itself. The UI marks the ones with an update available; `fluyt module catalog`
shows the same thing as a list.

A module is a separate executable, verified against its signed manifest
immediately before every launch. One that fails verification does not run, and
Fluyt keeps shipping syslog without it.

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

## Reporting a problem

Open an issue with the output of `fluyt version` and `fluyt status`, and say
which platform you are on. Please do not attach logs without reading them first;
they contain whatever your devices sent.
