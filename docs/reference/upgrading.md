---
slug: /reference/operations/upgrading
title: Upgrading fluxrig
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Upgrading fluxrig

This page lists what you do to move a running fleet from one release to the next. Each section covers one step between two releases. The [changelog](../changelog.md) explains each change. This page gives the order in which you act on them.

The page starts at v0.13. For an upgrade from an earlier release, read the BREAKING entries of each release you cross in the changelog.

## From v0.12 to v0.13 {#v012-to-v013}

Do these steps in the order of this section. Later steps need the earlier ones: the scenarios, for example, are imported only after the Mixer runs and the spec is uploaded.

### Record what the Mixer will forget

The registry moved from the Mixer's database file to the bus, and nothing is migrated. After the upgrade, the Mixer does not know your Racks or your scenarios. Before you start, record what you will re-create:

- The name of each Rack: `fluxrig admin racks list`.
- The scenario files that you run.

### Check the configuration

A key that maps to no setting, for example a misspelled one, now stops the Mixer or the Rack at startup. A duration that does not parse, for example `"5 seconds"`, also stops it, and the error names the field. Before, both were ignored, and the process ran with a default. Read your configuration files for keys and durations like these before you upgrade. Do not start the new Mixer to test its configuration: it migrates the registry when it starts, and the Racks must be upgraded first. In the upgrade step below, a process that refuses to start names the key or the field. Correct it and start the process again.

### Give the Mixer its API

The Mixer refuses to start when `api.auth_token` is not set, and it stops when its API stops. Set `api.auth_token`. For a test setup only, you can set `api.auth_disabled_dangerously = true` instead.

### Upgrade every Rack first, then the Mixer

The order matters. A v0.13 Rack works under a v0.12 Mixer. A v0.12 Rack under a v0.13 Mixer stops authenticating, because the new Mixer does not send the Rack's secret again.

1. Upgrade every Rack. Each one keeps its secret in a file beside its passport (`base.state_file` with a `.secret` suffix).
2. Upgrade the Mixer. It starts with an empty registry, and the upgrade deletes the old Rack, Mixer and Snake rows from its database file.
3. Each Rack enrolls again by itself. The Mixer logs `Registration denied: bootstrap secret mismatch`, and the Rack logs `Registration denied; dropping cached identity`. The Rack then enrolls with its `rack.bootstrap_secret`. You do not change anything on the Rack if its `rack.bootstrap_secret` matches the Mixer's `enrollment.bootstrap_secret`. Both default to `fluxrig`, so if you changed one of them, make them match.
4. Each Rack comes back with a new MachineID. It waits in `pending` unless `enrollment.auto_adopt` is on. Approve each Rack with the name you recorded: `fluxrig admin racks approve <id> --name <name>`.

Telemetry from before and after the upgrade carries different entity IDs.

### Move to the reference spec 3.0.0

The reference spec `iso8583-v87-ascii` now packs a standard ISO 8583 bitmap, and its version is `3.0.0`. The spec store keeps every version unchanged, so a Mixer that holds `v2.2.0` keeps serving the old layout under that name.

- Upload the new spec: `fluxrig spec import --api examples/specs/iso8583-v87-ascii.yaml`.
- Change each `iso8583-v87-ascii:v2.2.0` reference in your scenarios and translation specs to `iso8583-v87-ascii:v3.0.0`.
- A peer that you built against the old layout must change with it. The new layout has a primary bitmap of 16 characters. It has a secondary bitmap only when a field above 64 is present.

### Update a scenario based on the roaming example

If a scenario comes from the roaming enrichment example, set `await_store: true` on its correlation store. Its reply needs the stored entry to find its connection. A failed or slow write now fails the authorization, bounded by `store_timeout`.

### Import and activate each scenario

Import each scenario again, with the changes from the spec and roaming steps, and activate it: `fluxrig scenario import --api --activate <file>`. From now on, a restarted Mixer restores its active scenario by itself.

### Check afterwards

`fluxrig admin racks list` shows every Rack as `active` and `online`. A Rack that is still not online after `registry.liveness_ttl` (90 s by default) is not heartbeating. Read its log for the two `Registration denied` lines of the Rack and Mixer upgrade.
