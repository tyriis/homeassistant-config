---
date: 2026-07-29
topic: auto-block duration duplication (H1)
status: proposed
---

# Auto-Block Duration Duplication — Design

## 1. Problem

`automation/auto_block_control.yaml` lines 38–113 expand one logical operation
("start a lock + timer for the duration shown in `input_select.auto_block_duration`")
into five near-identical `choose` branches (~75 lines of YAML). Only two values
vary per branch: the lock `duration` (`15m`/`30m`/`45m`/`1h`/`2h`) and the
`timer.start` `duration` (`HH:MM:SS`).

Adding a new duration option today requires:

1. editing `inputs/select/auto_block_duration.yaml` to add the option, **and**
2. adding a sixth 11-line `choose` branch in the automation, **and**
3. keeping the two duration formats in sync by hand.

Three edits, two of them pure boilerplate, with no compile-time check that the
formats match the option label.

## 2. Constraints

- **Dashboard consumer.** `ui-lovelace/00-test.yaml:170` renders
  `input_select.auto_block_duration` via `custom:mushroom-select-card`. Any
  approach that changes the helper's entity_id or domain forces a dashboard
  edit and a `safe-refactoring` pass (entity rename → consumers).
- **Curated option set.** The five options are non-uniform
  (15, 30, 45, 60, 120 min). They are a deliberate UX choice, not a slider
  range — `45 min` and `2 hours` are in, `75/90/105 min` are out.
- **Two target formats.** `rest_command.create_lock` wants `15m`/`1h`/`2h`
  (Go-style duration); `timer.start` wants `HH:MM:SS`.
- **Codebase convention.** The same file already uses `condition: template`
  for the lock-owner check (lines 22–24, 124–126), and the repo's other office
  automations use Jinja2 templates. Templates are an established pattern here.
- **Skill guidance.** `home-assistant-best-practices` says prefer native
  conditions/helpers over templates — *but* explicitly endorses templates for
  **dynamic service data** (`template-guidelines.md` §1), which is exactly
  this case. The duplication is in service *data*, not in a condition.

## 3. Approaches

### Approach A — Template mapping (single dict + derived formats)

Replace the five `choose` branches with two service calls whose `duration`
fields are templates. Use **one** dict to map the select label → total minutes,
then derive both target formats from that single source of truth.

```yaml
sequence:
  - variables:
      minutes: >
        {% set m = {"15 min": 15, "30 min": 30, "45 min": 45,
                     "1 hour": 60, "2 hours": 120} %}
        {{ m.get(states('input_select.auto_block_duration'), 60) }}
  - service: rest_command.create_lock
    data:
      name: office_light
      owner: homeassistant-auto-block
      duration: >
        {{ (minutes // 60 ~ 'h') if minutes % 60 == 0 and minutes >= 60
           else (minutes ~ 'm') }}
  - service: timer.start
    data:
      duration: "{{ '%02d:%02d:00' | format(minutes // 60, minutes % 60) }}"
    target:
      entity_id: timer.auto_block
```

~12 lines replace ~75.

### Approach B — `input_number` helper instead of select

Replace `input_select.auto_block_duration` with `input_number.auto_block_minutes`
(min 15, max 120, step 15). Derive both formats from the numeric state.

- **Pro:** no dict; adding a duration needs no template edit (just widen the
  range). Numeric state is always present → no "unknown option" failure mode.
- **Con (decisive):** breaks the curated option set. Step 15 introduces
  75/90/105 min which were deliberately excluded. To preserve the exact five
  options you'd need step 15 *plus* a constraint, which `input_number` cannot
  express — you'd be back to validating the value, i.e. reintroducing a
  mapping. Non-uniform steps (15,15,15,60,60) aren't representable at all.
- **Con:** entity_id migration → dashboard `mushroom-select-card` must become a
  number/slider card; `safe-refactoring` workflow required. Higher blast
  radius for a cosmetic refactor.
- **Con:** the `1h`/`2h` vs `15m` formatting logic is fiddly and now runs for
  *every* value, not just the curated five.

### Approach C — One `input_boolean` per duration

Rejected. Five booleans instead of one select multiplies state and does not
reduce the `choose` branches — you still need a branch per boolean. Net
duplication increase.

### Approach D — Extract to a script

Move the five branches into `script.auto_block_start` with a `field` for the
duration label. The script body still needs the mapping, so this only
*relocates* the duplication unless combined with A. Justified only if multiple
automations call the same logic — here exactly one caller does. YAGNI.

## 4. Evaluation Matrix

| Criterion                         | A — Template (single dict) | B — input_number | C — booleans | D — script |
|-----------------------------------|----------------------------|------------------|--------------|------------|
| YAML LoC (automation)             | ~12 (−63)                  | ~12 (−63)        | ~+20         | ~12 here, +30 in script |
| Edits to add a duration           | 2 (select + 1 dict entry)  | 1 (range)        | 3+           | 2–3        |
| Load-time validation              | none (runtime template)    | none             | partial      | none       |
| Preserves curated option set     | **yes**                    | no               | yes          | yes        |
| Dashboard impact                  | none                       | card swap + rename | none       | none       |
| Entity-id migration / blast radius| none                       | medium           | low          | none       |
| Error handling on bad state       | dict `.get()` default      | n/a (always numeric) | n/a     | depends    |
| Single source of truth for formats| **yes** (minutes → both)   | yes              | no           | if A inside |
| Consistency with codebase         | matches existing templates | new pattern      | new pattern  | matches if templated |
| Skill alignment                   | endorsed use (dynamic svc data) | neutral     | neutral      | neutral    |

## 5. Recommendation

**Approach A**, with the single-dict refinement (map label → minutes once,
derive both target formats from `minutes`).

### Rationale

1. **Targets the actual duplication.** The boilerplate is in service *data*,
   and templates for dynamic service data are the explicitly-approved use case
   in `template-guidelines.md` §1. The skill's "prefer native" rule applies to
   *conditions and helpers*, not to service-call arguments.
2. **Single source of truth.** Mapping the label to `minutes` once — then
   formatting for the lock and the timer from that one number — eliminates the
   "two dicts to keep in sync" hazard that the raw Approach A in the review
   prompt has. Adding `"3 hours": 180` is one dict entry; both formats update
   automatically.
3. **Zero blast radius.** No entity rename, no dashboard edit, no
   `safe-refactoring` pass. The `mushroom-select-card` keeps working
   unchanged. This is a pure automation-internal refactor.
4. **Preserves the curated UX.** The five non-uniform options stay exactly as
   designed — Approach B cannot represent them without reintroducing a mapping.
5. **Consistent with the file.** The automation already uses
   `condition: template` for the owner check two branches above; introducing a
   template here is not a new pattern for the file.
6. **Honest error handling.** `dict.get(state, 60)` falls back to the helper's
   `initial` value (1 hour) if the select is `unknown`/`unavailable` or holds
   a label not in the dict. The automation fails soft, not hard.

### Why not B

B's only real advantage (no dict, always-numeric state) is outweighed by the
loss of the curated option set and the dashboard/entity migration cost. If the
option set were uniform (e.g. 15,30,45,60,75,90,105,120) B would be attractive;
for five hand-picked non-uniform values it is the wrong tool.

### Why not D standalone

D is a *container* decision, not a *duplication-elimination* decision. It only
removes duplication when combined with A. With a single caller, the extra
script indirection buys nothing. Revisit if a second consumer of this logic
appears.

## 6. Failure Modes & Mitigations

| Failure                              | Behavior under A                          | Mitigation |
|--------------------------------------|-------------------------------------------|------------|
| Select state `unknown`/`unavailable` | `dict.get` returns default `60`           | Acceptable; matches helper `initial` |
| Select label not in dict (drift)     | Same — default `60`                        | Add a log? `{{ log(..., 'warning') }}` optional |
| `rest_command.create_lock` HTTP fail | `timer.start` still runs → timer without lock | Pre-existing; out of scope. Consider `continue_on_error: false` + ordering in a later task |
| Template typo in format string       | Fails at runtime, not load                | Covered by `check_config` for syntax only; add an HA `test` run after edit |

## 7. Implementation Plan (for Fixer)

1. In `automation/auto_block_control.yaml`, replace the inner `choose:` block
   (lines 38–113, the five duration branches) with the `variables` + two
   service calls from §3.
2. Keep the outer `choose` structure (the `auto_block_toggle on` branch) intact
   — only its `sequence` changes.
3. Do **not** touch `inputs/select/auto_block_duration.yaml` or
   `ui-lovelace/00-test.yaml`.
4. Validate: `ha config check` (or equivalent) for YAML/schema; then manually
   toggle `input_boolean.auto_block` with each of the five select options and
   confirm (a) the lock is created with the right `duration` and (b)
   `timer.auto_block` shows the right countdown.
5. Verify the `off`/`timer_finished` branches are unchanged.

## 8. Out of Scope

- Lock-service HTTP failure handling (pre-existing; track separately).
- Generalizing to a script (Approach D) — defer until a second caller exists.
- Migrating the select to `input_number` — rejected per §5.