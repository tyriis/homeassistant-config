# Climate Cooling Script Consolidation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidate 3 nearly-identical `cool_<zone>` scripts into a single parameterized `cool_room` script.

**Architecture:** One script with a `zone` field (select: bedroom/livingroom/office) that maps to entity IDs via template variables, replacing the 3 per-zone copies. The 3 calling automations update their action target.

**Tech Stack:** Home Assistant YAML script configuration, `fields:` for parameters, Jinja2 templates for entity resolution.

**Spec:** `docs/superpowers/specs/2026-07-28-climate-cooling-script-design.md`

---

### Task 1: Create consolidated `cool_room` script

**Files:**
- Create: `scripts/cool_room.yaml`

- [ ] **Create the file**

```yaml
---
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

- [ ] **Commit**

```bash
git add scripts/cool_room.yaml
git commit -m "feat(script): add consolidated cool_room script with zone parameter"
```

---

### Task 2: Update `climate_bedroom_auto_on.yaml`

**Files:**
- Modify: `automation/climate_bedroom_auto_on.yaml`

- [ ] **Change the action target**

Replace line 41:
```yaml
  - action: script.cool_bedroom
```
with:
```yaml
  - action: script.cool_room
    data:
      zone: bedroom
      reason: hot day and bedroom temperature to high
```

- [ ] **Commit**

```bash
git add automation/climate_bedroom_auto_on.yaml
git commit -m "fix(automation): use cool_room script for bedroom AC"
```

---

### Task 3: Update `climate_livingroom_auto_on.yaml`

**Files:**
- Modify: `automation/climate_livingroom_auto_on.yaml`

- [ ] **Change the action target**

Replace line 43:
```yaml
  - action: script.cool_livingroom
```
with:
```yaml
  - action: script.cool_room
    data:
      zone: livingroom
      reason: hot day and livingroom temperature to high
```

- [ ] **Commit**

```bash
git add automation/climate_livingroom_auto_on.yaml
git commit -m "fix(automation): use cool_room script for livingroom AC"
```

---

### Task 4: Update `climate_office_auto_on.yaml`

**Files:**
- Modify: `automation/climate_office_auto_on.yaml`

- [ ] **Change the action target**

Replace line 27:
```yaml
  - action: script.cool_office
```
with:
```yaml
  - action: script.cool_room
    data:
      zone: office
      reason: hot day and office temperature to high
```

- [ ] **Commit**

```bash
git add automation/climate_office_auto_on.yaml
git commit -m "fix(automation): use cool_room script for office AC"
```

---

### Task 5: Remove old per-zone scripts

**Files:**
- Delete: `scripts/cool_bedroom.yaml`
- Delete: `scripts/cool_livingroom.yaml`
- Delete: `scripts/cool_office.yaml`

- [ ] **Delete the files**

```bash
git rm scripts/cool_bedroom.yaml scripts/cool_livingroom.yaml scripts/cool_office.yaml
```

- [ ] **Commit**

```bash
git commit -m "refactor(script): remove per-zone cool scripts, replaced by consolidated cool_room"
```

---

### Task 6: Final verification

- [ ] **Check git status**

```bash
git status
```
Expected: clean working tree on `feature/climate-cooling-script`.

- [ ] **Show final diff**

```bash
git diff main --stat
```
Expected: 1 new file, 3 modified, 3 deleted, ~150 lines net reduction.
