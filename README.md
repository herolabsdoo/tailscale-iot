# tailscale-iot

Patched builds of the [Tailscale](https://github.com/tailscale/tailscale) client and the
[headscale](https://github.com/juanfont/headscale) control server for IoT devices on slow or
metered cellular links.

Static Linux binaries for `armv6`, `mipsle` and `amd64` are published on the
[Releases](../../releases) page. Every release is built from a tag of this repository by the
included GitHub Actions workflow, from pinned upstream sources plus the patches in `patches/`.
This repository contains the build definition and the patches only; it carries no deployment
configuration.

## Motivation

Tailscale and headscale are tuned for well-connected hosts. On a 2G or congested LTE link, where a
round trip immediately after a re-dial can take 0.7–3.5 s, the default deadlines are not met:
every reconnect fails and is retried, each retry triggers a network check, and the resulting
traffic is paid for by the byte. On the control server, any node's state change is fanned out to
every connected node, including nodes that are not permitted to see it under the ACL policy; those
receive an empty envelope. On a metered link this background traffic dominates.

The patches address both sides. Measured per node per day on a 2G link, control-plane traffic
drops from ~10.4 MB (stock client and server) to 7.6 MB with the client patches and to ~2.5–3 MB
with the server patches as well.

## Patches

Each patch is one file per concern; every changed line carries a `tailscale-iot:` comment. Client
and server patches are independent, and any combination runs.

### Client (`patches/tailscale/`)

| Patch | Change | Rationale | Effect per node per day |
|---|---|---|---|
| `001-slow-link-timeouts` | DERP dial 1.5 → 15 s, DERP connect 10 → 90 s, STUN 3 → 15 s, netcheck 5 → 30 s and full report 5 min → 1 h, control watchdog 2 → 5 min | Stock deadlines are shorter than a 2G round trip after a re-dial, so every reconnect fails and is retried | Reconnect storms 3.5 → 0.5 MB (001 and 002 together, measured on one 2G node); ~190 forced re-logins eliminated |
| `002-no-netcheck-storms` | No netcheck on a broken DERP connection; STUN-only probes; DERP reconnect backoff up to 60 s | ~2,000 netchecks a day, each an ICMP probe and a TLS handshake with no benefit | Included in the figure above; netchecks reach zero after the first day |
| `003-cheaper-connections` | TLS session resumption for all dials; TCP keepalive 30 → 60 s | A resumed session is ~1 kB instead of ~5 kB and one round trip shorter; application-level keepalives already exist | ~0.2 MB of keepalive probes plus ~4 kB per reconnect (estimate); not visible on LTE |

### Control server (`patches/headscale/`)

| Patch | Change | Rationale | Effect per node per day |
|---|---|---|---|
| `001-no-empty-frames` | No MapResponse is sent when nothing visible to the node remains in it | Since headscale 0.29 the mapper filters content by ACL but still writes the empty envelope: 12–19k frames a day per node | 3.7 MB → ~0 |
| `002-tcp-keepalive` | Keepalive idle 75 s on the client-facing listener instead of Go's 15 s default | 5,800 probes a day per connection, redundant with the map poll's own keepalive | 1.2 MB → ~0 |

Every hunk is documented in [`patches/tailscale/README.md`](patches/tailscale/README.md) and
[`patches/headscale/README.md`](patches/headscale/README.md).

Runtime tuning is deliberately left out of the binaries: `GOGC`, `GOMEMLIMIT`,
`TS_DEBUG_DISABLE_PORTLIST` and `TS_DEBUG_RESTUN_STOP_ON_IDLE` are upstream knobs, set by
whatever starts the daemon. The port scanner is not compiled in (`portlist` is not part of the
feature set).

## Building

```sh
git submodule update --init      # tailscale and headscale
./build                          # all targets
./build tailscale/mipsle         # one target
```

| Target | Toolchain flags | Output |
|---|---|---|
| `tailscale/armv6` | `GOARCH=arm GOARM=5` — ARM11-class SoCs without an FPU | `tailscaled`, ~15 MB |
| `tailscale/mipsle` | `GOARCH=mipsle GOMIPS=softfloat` — MIPS32 SoCs without an FPU | `tailscaled`, ~17 MB |
| `tailscale/amd64` | `GOARCH=amd64` | `tailscaled`, ~16 MB |
| `headscale/amd64` | `GOARCH=amd64` | `headscale`, ~50 MB |

Requirements: `git`, `patch`, `curl`. The tailscale build uses upstream's own toolchain
(`tool/go` downloads the pinned Go into `~/.cache/tsgo` on first use) with the minimal feature set
plus `osrouter`, `iptables`, `debug`, `sdnotify`, `bakedroots`, `unixsocketidentity`,
`clientmetrics` and `ipnbus`. The CLI is embedded in `tailscaled`; `tailscale` is a symlink to it
and should be copied as a regular file on filesystems without symlink support. headscale is built
with the stock Go version named in its `go.mod` (downloaded into `work/go/`) from a clone, so
`headscale version` reports `v<VERSION>+dirty`, the suffix denoting the patches. Output is written
to `targets/<program>/<arch>/`.

Builds are reproducible: `-trimpath`, no VCS stamp and a pinned toolchain yield identical bytes
on any machine, so a release asset can be verified against a local build by SHA-256.

## Releases

A release is created by pushing the tag `v<VERSION>`, where `VERSION` is the file at the root of
this repository. The workflow builds every target and attaches:

| Asset | Content |
|---|---|
| `<program>-v<upstream>_iot-v<VERSION>_<arch>.gz` | the binary, gzip-compressed — e.g. `tailscale-v1.102.4_iot-v1.0_mipsle.gz` |
| `<asset>.sha256` | SHA-256 of the uncompressed binary |
| `LICENSE` | this repository's license (BSD 3-Clause) |
| `LICENSE.tailscale`, `PATENTS.tailscale`, `LICENSE.headscale` | upstream license texts and patent grant |

The workflow refuses a tag that does not match `VERSION`. Any change that alters the produced
bytes — a patch, an upstream bump, a feature flag — requires a new `VERSION`; an asset name is
never reused for different content.

## Updating the upstream pin

1. Fetch tags in the submodule (`git -C tailscale fetch --tags`, or `headscale`), check out the
   new tag, and write the version into `targets/<program>/VERSION`.
2. Run `./build <target>`. A patch that no longer applies stops the build at that point; update it
   in `patches/` (for tailscale, regenerate from a scratch checkout of the new tag; for headscale,
   the clone in `work/headscale-amd64/` is a git repository — edit and `git diff`).
3. Increment `VERSION`; commit the submodule pointer, both version files and the patches together;
   tag `v<VERSION>`; push the branch and the tag.

headscale 0.29.x requires clients of version 1.80.0 or later (capability version 113). The client
patches are maintained against tailscale 1.102.x.

## Deploying the patched headscale

`targets/headscale/amd64/headscale` (or the release asset) is the upstream program with the two
patches applied. It replaces the binary inside the official image; configuration, database, state,
the embedded DERP server and ACME are unaffected, no configuration key is added, and no migration
runs:

```dockerfile
FROM docker.io/headscale/headscale:v0.29.3
COPY headscale /ko-app/headscale
```

`/ko-app/headscale` is the binary and entrypoint of the official distroless image. Rolling back
means returning to the stock image tag.

Patch 002 configures keepalive on headscale's own listener, so headscale must terminate the
client TCP connections itself; a TCP-terminating proxy in front of it would negate the patch. There
is no configuration-only substitute for patch 001: `tuning.batch_change_delay` delays empty frames
but does not remove them.

## Repository layout

```
VERSION                 version of this repository; the tag v<VERSION> produces a release
tailscale/              upstream tailscale/tailscale (submodule, pinned to targets/tailscale/VERSION)
headscale/              upstream juanfont/headscale (submodule, pinned to targets/headscale/VERSION)
patches/tailscale/      client patches and their documentation
patches/headscale/      control-server patches and their documentation
build                   build script
targets/<program>/      VERSION (upstream pin) and one output directory per architecture (ignored)
.github/workflows/      release.yml
```

## License

Tailscale and headscale are distributed under the BSD 3-Clause License (`tailscale/LICENSE`,
`tailscale/PATENTS`, `headscale/LICENSE`). The patches and build tooling in this repository are
derivative works of those projects and are provided under the same terms — see
[`LICENSE`](LICENSE). All of these files accompany every release. The software is provided "as
is", without warranty of any kind.
