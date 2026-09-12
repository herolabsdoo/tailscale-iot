# Control-server patches

Applied by `../../build` on top of tag `v<VERSION>` of `../../headscale/`, where `VERSION` is
`../../targets/headscale/VERSION`. Every changed line carries a `tailscale-iot:` comment. Both
patches concern deployments with many nodes on metered links, each holding one long-lived control
connection, where the ACL policy prevents most nodes from seeing one another.

## 001-no-empty-frames

`hscontrol/mapper/mapper.go`, `buildFromChange`: after the MapResponse has been built, no response
is returned when the change requested nothing else (no self node, DERP map, DNS, domain, policy,
ping or full map) and the built response carries no `PeersChanged`, `PeersChangedPatch` or
`PeersRemoved`. `handleNodeChange` already sends nothing for a nil response.

Rationale: any node's connect, disconnect, endpoint change or MapRequest is fanned out to every
connected node. Since 0.29 the mapper filters the content by ACL but still writes the empty
envelope (ControlTime only, ~300 B on the wire): 12–19k frames a day per node, 3.7 MB a day. The
decision is made on the built response rather than on a visibility count because upstream's
`buildTailPeers` is the authoritative reducer. Full updates are always sent; a full update with
zero peers carries `PeersRemoved`, which the client needs.

Tests: `TestBuildFromChangeFiltersPeerPatchesByVisibility` and
`TestBuildFromChangeFiltersUserProfilesByVisibility` now expect no response where they expected an
empty one; `TestBatcherWorkQueueBatching` expects 6 updates instead of 7, the seventh having been
the empty frame left over from the receiving node's own online change (a node is never its own
peer, so upstream already drops the patch itself). The package passes.

## 002-tcp-keepalive

`hscontrol/app.go`, `Serve()`: the client-facing listener is configured with
`net.KeepAliveConfig{Idle: 75 s, Interval: 15 s, Count: 3}`, and TLS is layered on top with
`tls.NewListener` instead of `tls.Listen`.

Rationale: Go's `net.Listen` probes every accepted socket every 15 s: 5,800 probes a day per
connection, and with replies and the second (DERP) connection about 1.2 MB a day per node. The map
long-poll writes its own keepalive every 50–59 s, so a healthy connection never reaches 75 s of
idle time; a dead one is detected in about 2 minutes instead of 2.5.

## Maintenance

Patches are applied with `patch -p1` on a clone of the pinned tag; one that no longer applies
stops the build. To regenerate after an upstream bump, use the clone in
`../../work/headscale-amd64/` (a git repository): resolve, then `git diff -- <files>` per patch.
