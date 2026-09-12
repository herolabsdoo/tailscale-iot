# Client patches

Applied by `../../build` on top of tag `v<VERSION>` of `../../tailscale/`, where `VERSION` is
`../../targets/tailscale/VERSION`. Every changed line carries a `tailscale-iot:` comment. All
three patches target links where the round trip immediately after a re-dial is 0.7–3.5 s, as on
2G; on a healthy LTE link they have no visible effect.

## 001-slow-link-timeouts — deadlines a slow link can meet

| File | Was → is | Rationale |
|---|---|---|
| `control/controlclient/direct.go` `watchdogTimeout` | 2 → 5 min | The control long-poll expects a keepalive per minute. One late keepalive on a loaded bearer ended the poll: a new TLS connection, a login (~66 kB) and a full netmap, ~190 times a day. |
| `derp/derphttp/derphttp_client.go` `connect` timeout | 10 → 90 s | DNS, TCP, TLS (a ~4.85 kB chain) and the DERP upgrade after a re-dial take longer than 10 s. |
| `derp/derphttp/derphttp_client.go` `dialNodeTimeout` | 1.5 → 15 s | A SYN round trip is 0.7–3.5 s; at 1.5 s every DERP dial failed. |
| `net/netcheck/netcheck.go` STUN and report timeouts | 3 → 15 s, 5 → 30 s | A missed STUN reply was interpreted as "UDP blocked", triggering the ICMP and HTTPS fallbacks (see 002). |
| `net/netcheck/netcheck.go` full report interval | 5 min → 1 h | With a single DERP region there is nothing to re-rank. |

## 002-no-netcheck-storms — stop the retry loop

| File | Change | Rationale |
|---|---|---|
| `wgengine/magicsock/derp.go` DERP reconnect backoff | max 5 → 60 s | A reconnect attempt every ~2 s amounted to 850–2,460 attempts a day. |
| `wgengine/magicsock/derp.go` `runDerpReader` | no `ReSTUN("derp-recv-error")` | Every broken DERP connection started a netcheck, ~2,000 a day, each an ICMP probe and a TLS handshake. Link changes trigger their own netcheck. |
| `wgengine/magicsock/magicsock.go` `updateNetInfo` | `OnlySTUN` unless in TCP-443-only mode | Carriers commonly drop ICMP, and the HTTPS fallback is a TLS handshake that yields nothing. |

## 003-cheaper-connections — pay less per reconnect

| File | Change | Rationale |
|---|---|---|
| `net/tlsdial/tlsdial.go` | one `tls.ClientSessionCache` for every dial | A resumed TLS session is ~1 kB instead of ~5 kB (the server's certificate chain) and one round trip shorter. `VerifyConnection` still runs. |
| `net/netknob/netknob.go` `PlatformTCPKeepAlive` | 30 → 60 s | Control and DERP have application-level keepalives; the client's own probes were 0.38 MB per day. |

This is the smallest of the three patches and the first to drop if a reduced set is wanted.

## Maintenance

Patches are applied with `patch -p1`; one that no longer applies stops the build. To regenerate
after an upstream bump: export the new tag (`git archive`), initialise a git repository in the
export, apply the patch, resolve, and `git diff`.
