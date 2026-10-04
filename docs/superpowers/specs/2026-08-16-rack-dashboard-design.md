# Rack Dashboard — Power Consumption & Overview

**Date:** 2026-08-16
**Status:** Approved
**Branch:** feature/rack-dashboard (proposed)

## Goal

Add a Rack dashboard view to the existing Lovelace dashboard showing live power draw per Nous A1Z cluster plug + combined cluster draw, daily and weekly kWh consumption per plug and for the cluster, plus a small rack overview (Govee LED light, rack motion sensor, status.techtales.io). Built with native HA helpers (min_max sum + utility_meter) in YAML, following repo conventions, so more metered rack devices can be added later.

## Current State

- The rack (server/network rack, HA area "Rack") is powered via three Nous A1Z smart plugs: `nous a1z cluster node01/02/03` (switches all currently `on`). Each exposes live power (W) and accumulated energy (kWh, total_increasing) sensors in HA, e.g. `sensor.nous_a1z_cluster_node01_power` and `sensor.nous_a1z_cluster_node01_energy` (exact energy entity IDs to be verified against the live instance at implementation time — they live in .storage, not in git).
- The repo has NO utility_meter integration, NO min_max aggregation, and NO rack power/energy entities in git yet. Existing power visualization is limited to `graph: line / type: sensor` cards in 04-basement.yaml and 03-office.yaml.
- The dashboard is YAML-mode: `configuration.yaml` → `lovelace: dashboards: lovelave-test` with `views: !include_dir_list ui-lovelace/` — each file in `ui-lovelace/` is one view. Free view slots include `07-` (05, 07, 10 are free).
- Repo include conventions: `counter: !include_dir_merge_named counters/` (each file `---` + named key + config), `sensor: !include_dir_list sensors/` (each file `---` + `platform: X` flat config), `timer: !include_dir_merge_named timers/`.

## Design

### Updated file: `configuration.yaml`

Add to `configuration.yaml` (in the helper includes block, after `timer:`):

```yaml
utility_meter: !include_dir_merge_named utility_meters/
```

### New files: `utility_meters/` (8 files, each named-key merged, matching counters/ pattern)

| File | Key | Source | Cycle |
|---|---|---|---|
| `rack_cluster_daily.yaml` | `rack_cluster_daily` | `sensor.rack_cluster_energy` | daily |
| `rack_cluster_weekly.yaml` | `rack_cluster_weekly` | `sensor.rack_cluster_energy` | weekly |
| `nous_node01_daily.yaml` | `nous_node01_daily` | `sensor.nous_a1z_cluster_node01_energy` | daily |
| `nous_node01_weekly.yaml` | `nous_node01_weekly` | `sensor.nous_a1z_cluster_node01_energy` | weekly |
| `nous_node02_daily.yaml` | `nous_node02_daily` | `sensor.nous_a1z_cluster_node02_energy` | daily |
| `nous_node02_weekly.yaml` | `nous_node02_weekly` | `sensor.nous_a1z_cluster_node02_energy` | weekly |
| `nous_node03_daily.yaml` | `nous_node03_daily` | `sensor.nous_a1z_cluster_node03_energy` | daily |
| `nous_node03_weekly.yaml` | `nous_node03_weekly` | `sensor.nous_a1z_cluster_node03_energy` | weekly |

Example file content (each file is `---` + the single named key):

```yaml
---
rack_cluster_daily:
  source: sensor.rack_cluster_energy
  cycle: daily
```

### New files: `sensors/` (2 files, flat platform config, matching sensors/ pattern)

`sensors/rack_cluster_power.yaml`:

```yaml
---
platform: min_max
name: Rack Cluster Power
unique_id: rack_cluster_power
type: sum
entity_ids:
  - sensor.nous_a1z_cluster_node01_power
  - sensor.nous_a1z_cluster_node02_power
  - sensor.nous_a1z_cluster_node03_power
unit_of_measurement: W
device_class: power
```

`sensors/rack_cluster_energy.yaml`:

```yaml
---
platform: min_max
name: Rack Cluster Energy
unique_id: rack_cluster_energy
type: sum
entity_ids:
  - sensor.nous_a1z_cluster_node01_energy
  - sensor.nous_a1z_cluster_node02_energy
  - sensor.nous_a1z_cluster_node03_energy
unit_of_measurement: kWh
device_class: energy
state_class: total_increasing
```

### New file: `ui-lovelace/07-rack.yaml` (one view, `type: sections`)

Frontmatter: `title: Rack`, `path: rack`, `icon: mdi:server-network`, `type: sections`, `max_columns: 4`, empty `badges`/`cards` like sibling views.

Sections (each a `type: grid` containing a `type: heading` card + cards, matching 01-home.yaml / 04-basement.yaml house style):

1. **Power** — heading `Power` (icon mdi:lightning-bolt); tiles for `sensor.rack_cluster_power` (name: Cluster), `sensor.nous_a1z_cluster_node01_power` (Node 01), `_node02_` (Node 02), `_node03_` (Node 03); a `custom:mini-graph-card` of `sensor.rack_cluster_power` (24h, matching the heating-graph style in 01-home.yaml: hour24, smoothing, extrema, no labels).
2. **Daily** — heading `Daily` (icon mdi:calendar-today); tiles for `utility_meter.rack_cluster_daily`, `utility_meter.nous_node01_daily`, `_node02_daily`, `_node03_daily`; a native `statistics-graph` card (`entity: sensor.rack_cluster_energy`, `period: day`, `days_to_show: 14`, `stat_types: [change]`, `chart_type: bar`) to show last 14 days of cluster consumption.
3. **Weekly** — heading `Weekly` (icon mdi:calendar-week); tiles for `utility_meter.rack_cluster_weekly`, `utility_meter.nous_node01_weekly`, `_node02_weekly`, `_node03_weekly`.
4. **Rack** — heading `Rack` (icon mdi:server-network); tiles for `light.govee_light_rack` (Govee LED), `binary_sensor.mijia_motion_rack_occupancy` (motion, state_content state + last_updated), and the status monitor entity (entity id to verify at implementation — currently referenced as `sensor.status_techtales_io` in 00-test.yaml; verify against live instance, e.g. could be `switch.status_techtales_io`).

### Error handling

min_max `sum` goes `unknown` if any source is unavailable — the dashboard shows the cluster tile as unavailable rather than silently under-reporting (accepted trade-off); per-plug tiles remain independent.

### What does NOT change

- No new dashboard registration in configuration.yaml `lovelace:` block (view is added to existing `lovelave-test` dashboard via ui-lovelace/ dir)
- No automations, scripts, scenes, groups, or existing views are touched
- No `.storage/` files are edited; helpers are versioned in git as YAML

## Migration

1. Verify exact entity IDs against live instance (HA UI → Settings → Devices & Services → Nous A1Z cluster plugs): the 3 power sensors, 3 energy sensors, and the status.techtales.io entity id. Confirm energy sensors are `total_increasing` kWh.
2. Add `utility_meter: !include_dir_merge_named utility_meters/` to configuration.yaml
3. Create 8 files in `utility_meters/`
4. Create 2 files in `sensors/`
5. Create `ui-lovelace/07-rack.yaml`
6. Validate: run `yamllint` on changed files (pre-commit), then `python -m homeassistant --config . --script check_config` (same as repo CI)
7. Commit with conventional commit message `feat(lovelace): add rack dashboard with power consumption`