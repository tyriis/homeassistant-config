# Climate Cooling Script Consolidation

**Date:** 2026-07-28
**Status:** Approved
**Branch:** feature/climate-cooling-script

## Goal

Consolidate 3 nearly-identical `cool_<zone>` scripts (bedroom, livingroom, office) into a single parameterized script to eliminate duplication and simplify maintenance.

## Current State

Three scripts at `scripts/cool_bedroom.yaml`, `scripts/cool_livingroom.yaml`, `scripts/cool_office.yaml` — each 52 lines, 95% identical. Only differences are entity names (e.g., `climate.bedroom` vs `climate.livingroom`, lock names `hisense_ac_bedroom` vs `hisense_ac_livingroom`).

Each script does:
1. Release existing lock via REST API
2. Create 20-minute automation lock via REST API
3. If AC not in cooling mode → set HVAC mode to cool
4. If AC is off → turn it on
5. Log to system_log

## Design

### New file: `scripts/cool_room.yaml`

A single parameterized script with a `zone` selector field:

```yaml
cool_room:
  alias: Cool Room
  mode: single
  fields:
    zone:
      required: true
      selector:
        select:
          options:
            - bedroom
            - livingroom
            - office
    reason:
      selector:
        text:
  sequence:
    - variables:
        lock_name: "hisense_ac_{{ zone }}"
        ac_entity: "climate.{{ zone }}"
    - service: rest_command.release_lock
      data:
        name: "{{ lock_name }}"
    - service: rest_command.create_lock
      data:
        name: "{{ lock_name }}"
        owner: homeassistant-automation
        duration: 20m
    - if:
        - condition: template
          value_template: "{{ states(ac_entity) != 'cool' }}"
      then:
        - service: climate.set_hvac_mode
          target:
            entity_id: "{{ ac_entity }}"
          data:
            hvac_mode: cool
        - service: system_log.write
          data:
            level: info
            message: "Set {{ zone }} AC mode: cooling."
            logger: homeassistant.components.script.cool_room
    - if:
        - condition: template
          value_template: "{{ states(ac_entity) == 'off' }}"
      then:
        - service: climate.turn_on
          target:
            entity_id: "{{ ac_entity }}"
        - service: system_log.write
          data:
            level: info
            message: "Started {{ zone }} AC: {{ reason | default('Manual') }}."
            logger: homeassistant.components.script.cool_room
```

### Updated files

Three `climate_*_auto_on.yaml` automations — change the action target:

| File | Old action | New action |
|---|---|---|
| `climate_bedroom_auto_on.yaml` | `script.cool_bedroom` | `script.cool_room { zone: "bedroom" }` |
| `climate_livingroom_auto_on.yaml` | `script.cool_livingroom` | `script.cool_room { zone: "livingroom" }` |
| `climate_office_auto_on.yaml` | `script.cool_office` | `script.cool_room { zone: "office" }` |

### Deleted files

- `scripts/cool_bedroom.yaml`
- `scripts/cool_livingroom.yaml`
- `scripts/cool_office.yaml`

### Entity naming convention

The template mapping relies on the consistent naming pattern across all zones:
- Climate entity: `climate.<zone>` (e.g., `climate.bedroom`)
- Lock name: `hisense_ac_<zone>` (e.g., `hisense_ac_bedroom`)

This holds true for all 3 current zones (bedroom, livingroom, office).

### What does NOT change

- The `climate_*_auto_on.yaml` triggers and conditions remain untouched
- No sensor, helper, input_datetime, or lock service changes
- No automation behavior changes — strictly a refactor

## Migration

1. Create `scripts/cool_room.yaml`
2. Update 3x `climate_*_auto_on.yaml` to call new script
3. Delete 3x old `cool_<zone>.yaml` scripts
4. Validate with `ha core check`
