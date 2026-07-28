# Lock Collision Bug (C1) — Auto Block Control vs Manual Override

**Date:** 2026-07-28
**Status:** Draft — awaiting review
**Related PR:** #150 (`automation/auto_block_control.yaml`)

## Problem Summary

`automation/auto_block_control.yaml` (PR #150) has two branches — `timer_finished` and the `off` branch — that call `rest_command.release_lock` **unconditionally**. During the auto-block period, `office_light_manual_control.yaml` can replace the lock owner from `homeassistant-auto-block` to `homeassistant-<user_id>`. When the timer expires or auto-block is turned off, the unconditional `release_lock` wipes the user's manual override lock prematurely.

A third unconditional `release_lock` in the `on` branch (pre-create) and the `ha_start` trigger compound the problem: every HA restart with `input_boolean.auto_block` off wipes **any** existing lock (manual, gaming, etc.), not just auto-block locks.

## System Architecture

### Locking Service

External REST service at `http://locking-service:3000`:

| Endpoint | Method | Behavior |
|---|---|---|
| `/locks` | POST | Create/replace lock. Body: `{key, owner, duration}`. Only **one lock per key** exists at a time — POST replaces. |
| `/locks/{name}` | DELETE | Release/delete a lock. Unconditional — no owner check server-side. |
| `/locks/{name}` | GET | Return lock state (or empty body if unlocked). |

### HA REST Commands

- `rest_command.create_lock` → POST `/locks` with `{key: name, owner, duration}`
- `rest_command.release_lock` → DELETE `/locks/{name}`

### Sensor Layer

- `sensor.office_light_lock_status` — REST sensor polling `GET /locks/office_light` every **15 seconds**. Reports `locked`/`unlocked`/`unknown` with attributes `owner`, `expireAt`, `createdAt`, `duration`, `key`.
- `binary_sensor.office_light_automation_block_status` — template binary sensor. `on` when lock is `locked` AND owner != `homeassistant-automation`. This blocks the main light automation.

### Lock Owners

| Owner | Created by | Duration | Purpose |
|---|---|---|---|
| `homeassistant-<user_id>` | `office_light_manual_control.yaml` | 1h | Manual scene/light change |
| `homeassistant-gaming-purple` | `office_light_gaming_mode_purple.yaml` | 4h | Gaming mode |
| `homeassistant-auto-block` | `auto_block_control.yaml` (NEW) | variable (15m–2h) | UI toggle block |
| `homeassistant-automation` | Various scripts (`cool_room`, etc.) | 20m | Automation's own lock (NOT blocked) |

All owners except `homeassistant-automation` block the main light automation.

### Auto Block Control Automation

`auto_block_control.yaml`, `mode: restart`. Three triggers:

1. `auto_block_toggle` — state change of `input_boolean.auto_block`
2. `ha_start` — HA restart
3. `timer_finished` — `timer.auto_block` finishes

Three action branches (choose):

| Branch | Condition | Actions |
|---|---|---|
| `timer_finished` | trigger id == timer_finished | `release_lock` (unconditional), `input_boolean.turn_off` |
| `on` | `input_boolean.auto_block` == on | `release_lock` (unconditional), then `create_lock` (auto-block owner) + `timer.start` |
| `off` | `input_boolean.auto_block` == off | `release_lock` (unconditional), `timer.cancel` |

**Critical detail:** the `timer_finished` branch turns off `input_boolean.auto_block` (line 27-29), which re-triggers the `auto_block_toggle` trigger, firing the `off` branch. This causes a **double release** — the lock is deleted, then the `off` branch tries to delete it again (no-op or 404 error).

### Existing Owner-Check Pattern

`office_light_gaming_mode_purple_auto_off.yaml` already uses an owner check before releasing:

```yaml
conditions:
  - condition: template
    value_template: "{{ state_attr('sensor.office_light_lock_status', 'owner') == 'homeassistant-gaming-purple' }}"
```

This is the established pattern in this codebase for safe lock release.

---

## Scenario Analysis

### Scenario A — Normal auto-block, no manual intervention

**Flow:** User turns on auto-block → timer runs → timer expires.

**Current (no owner check):**
1. Auto-block on → `on` branch: release (no-op or wipes prior lock), create lock `homeassistant-auto-block`, start timer.
2. Timer expires → `timer_finished` branch: release lock. Lock deleted. `input_boolean.auto_block` turned off.
3. Turning off input_boolean re-triggers `off` branch: release again (no-op, lock already gone). Cancel timer (already finished).
- **Result:** Lock released, auto-block off. **Correct.**

**With owner check:**
1. Same as above.
2. Timer expires → `timer_finished` branch: owner == `homeassistant-auto-block`? Yes → release. Lock deleted. Turn off input_boolean.
3. Re-trigger `off` branch: owner check — sensor may still show `homeassistant-auto-block` (15s lag) or `unlocked` (owner None). If stale: passes check, tries DELETE on already-deleted lock (404, harmless with `continue_on-error`). If fresh: fails check, skips release.
- **Result:** Lock released, auto-block off. **Correct.** Minor: possible redundant DELETE that 404s.

### Scenario B — Manual change during auto-block, then timer expires

**Flow:** Auto-block on → user manually changes light → timer expires.

**Current (no owner check):**
1. Auto-block on → lock owner = `homeassistant-auto-block`, timer running.
2. User manually changes light → `office_light_manual_control.yaml` fires: release lock, create lock `homeassistant-<user_id>` (1h). **Owner is now `homeassistant-<user_id>`.** Timer is still running. `input_boolean.auto_block` is still on.
3. Timer expires → `timer_finished` branch: **unconditional release** wipes the user's manual lock. The 1h manual override is gone after only a fraction of its intended duration.
4. `input_boolean.auto_block` turned off → `off` branch: release again (no-op).
- **Result:** User's manual override destroyed. **BUG (C1).**

**With owner check:**
1-2. Same as above. Owner = `homeassistant-<user_id>`.
3. Timer expires → `timer_finished` branch: owner == `homeassistant-auto-block`? **No** (it's `homeassistant-<user_id>`) → **skip release**. User's manual lock survives. Turn off `input_boolean.auto_block`.
4. Re-trigger `off` branch: owner == `homeassistant-auto-block`? No → skip release. Cancel timer.
- **Result:** User's manual override preserved. Auto-block UI shows off (period ended). **Correct.** The manual lock expires naturally at its own 1h deadline.

### Scenario C — Manual change during auto-block, then user turns off auto-block

**Flow:** Auto-block on → user manually changes light → user turns off auto-block.

**Current (no owner check):**
1. Auto-block on → lock owner = `homeassistant-auto-block`, timer running.
2. User manually changes light → owner = `homeassistant-<user_id>`, 1h.
3. User turns off auto-block → `off` branch: **unconditional release** wipes user's manual lock. Cancel timer.
- **Result:** User's manual override destroyed. **BUG (C1).**

**With owner check:**
1-2. Same. Owner = `homeassistant-<user_id>`.
3. User turns off auto-block → `off` branch: owner == `homeassistant-auto-block`? **No** → **skip release**. Cancel timer.
- **Result:** User's manual override preserved. Auto-block is off. **Correct.**

### Scenario D — User turns off auto-block normally (no manual intervention)

**Flow:** Auto-block on → user turns off auto-block.

**Current (no owner check):**
1. Auto-block on → owner = `homeassistant-auto-block`, timer running.
2. User turns off auto-block → `off` branch: release lock. Cancel timer.
- **Result:** Lock released, block lifted immediately. **Correct.**

**With owner check:**
1. Same.
2. User turns off auto-block → `off` branch: owner == `homeassistant-auto-block`? Yes → release. Cancel timer.
- **Result:** Lock released, block lifted immediately. **Correct.**

### Scenario E — User turns on auto-block while a manual lock exists

**Flow:** Manual lock active (owner = `homeassistant-<user_id>`) → user turns on auto-block.

**Current (no owner check):**
1. Manual lock exists: owner = `homeassistant-<user_id>`, expires at T+1h.
2. User turns on auto-block → `on` branch: release lock (wipes manual lock), create lock `homeassistant-auto-block`, start timer.
- **Result:** Manual lock replaced by auto-block lock. The user's manual override is gone. **Arguably intentional** — the user explicitly turned on auto-block, so it should take effect. But the manual lock's remaining time is lost.

**With owner check on the release (line 36):**
1. Same.
2. User turns on auto-block → `on` branch: owner == `homeassistant-auto-block`? **No** (it's `homeassistant-<user_id>`) → skip release. **But `create_lock` (POST) still replaces the lock.** Manual lock is still replaced.
- **Result:** Same as current — manual lock replaced. The owner check on the pre-create release **does not help** because POST replaces regardless.

**Implication:** The owner check on the `on` branch release is ineffective for preserving manual locks. To actually preserve manual locks when turning on auto-block, we would need to check the owner **before `create_lock`** and skip the entire create if a non-auto-block lock exists. This is a **separate design decision** (should auto-block override manual locks?) and not part of the C1 bug. The current behavior (auto-block overrides) appears intentional.

**Recommendation for the `on` branch release:** Remove the pre-create `release_lock` entirely — it is redundant because `create_lock` (POST) replaces the existing lock. This eliminates a unnecessary DELETE call and simplifies the code. The `create_lock` will replace whatever lock exists, which is the current (intentional) behavior.

### Scenario F — HA restart during various states

The `ha_start` trigger fires the automation on every restart. The branch taken depends on `input_boolean.auto_block` state (which persists across restarts as an input_boolean).

**F1: Restart with auto-block on, no manual intervention**

**Current:**
1. HA restarts → `ha_start` trigger → `on` branch: release lock, create new auto-block lock, start timer.
2. Timer is reset (HA timers don't persist by default) — auto-block duration restarts from the selected duration, not the remaining time.
- **Result:** Lock replaced with fresh auto-block lock. Duration reset. **Minor issue** — the auto-block period is extended. Pre-existing, not C1.

**With owner check:** Same result — `create_lock` replaces regardless. The owner check on the pre-create release is ineffective (as in Scenario E).

**F2: Restart with auto-block on, manual lock had replaced auto-block lock**

**Current:**
1. Before restart: owner = `homeassistant-<user_id>` (manual lock replaced auto-block).
2. HA restarts → `on` branch: **unconditional release** wipes manual lock. Create new auto-block lock. Start timer.
- **Result:** User's manual override destroyed on restart. **BUG (variant of C1).**

**With owner check:**
1. Same.
2. `on` branch: owner == `homeassistant-auto-block`? No → skip release. But `create_lock` still replaces. Manual lock still replaced.
- **Result:** Same as current. The owner check on the `on` branch release doesn't help (POST replaces). **Not fixed by owner check on release.** Would require checking before `create_lock` (separate design decision).

**F3: Restart with auto-block off, manual lock exists**

**Current:**
1. Before restart: auto-block off, manual lock active (owner = `homeassistant-<user_id>`).
2. HA restarts → `off` branch: **unconditional release** wipes manual lock. Cancel timer.
- **Result:** **Every HA restart wipes ALL existing locks** (manual, gaming, etc.) when auto-block is off. **This is a broader bug than C1** — it affects gaming mode and any other lock consumer, not just manual overrides.

**With owner check:**
1. Same.
2. `off` branch: owner == `homeassistant-auto-block`? **No** (it's `homeassistant-<user_id>`) → **skip release**. Cancel timer.
- **Result:** Manual lock survives restart. **Correct.** This is a significant additional fix — the owner check prevents HA restart from wiping unrelated locks.

**F4: Restart with auto-block off, stale auto-block lock exists**

This can happen if auto-block was turned off but the lock wasn't released (e.g., due to a network error to the locking service).

**With owner check:**
1. Auto-block off, but lock owner = `homeassistant-auto-block` (stale).
2. HA restarts → `off` branch: owner == `homeassistant-auto-block`? **Yes** → release. Cancel timer.
- **Result:** Stale auto-block lock cleaned up. **Correct.**

---

## Edge Cases

### 1. Sensor unavailable or stale (15s polling lag)

`sensor.office_light_lock_status` polls every 15 seconds. There is a window where the sensor's reported owner lags behind the actual lock state.

**Race condition:** User manually changes light → `manual_control` creates lock with `homeassistant-<user_id>`. Within the next 15 seconds, the auto-block timer expires. The sensor still shows owner = `homeassistant-auto-block` (not yet polled). The owner check passes, and the manual lock is released. **The owner check fails to protect in this window.**

**Probability:** Low. The manual change and timer expiry must coincide within a 15-second window. The manual lock has a 1h duration; the timer has a 15m–2h duration. The overlap window is 15s out of the total auto-block period.

**Sensor unavailable:** If the locking service is down, the sensor reports `unknown` and `state_attr(..., 'owner')` returns `None`. The owner check `None == 'homeassistant-auto-block'` is `false` → release skipped. This is **fail-safe** (don't release if unsure). The lock will expire naturally on the server when its duration ends. Acceptable.

**Mitigation for the race condition:** See Alternative 1 (server-side owner-aware DELETE) below — it eliminates the race entirely by making the check atomic server-side.

### 2. The ON branch — should it also check before releasing?

The `on` branch (line 36) calls `release_lock` before `create_lock`. As analyzed in Scenario E, this release is **redundant** — `create_lock` (POST) replaces the existing lock regardless. The owner check on this release is ineffective for preserving manual locks because the subsequent POST replaces anyway.

**Recommendation:** Remove the pre-create `release_lock` from the `on` branch entirely. It serves no purpose (POST replaces) and adds a unnecessary network call. This also eliminates the double-release concern for the `on` branch.

If the design decision is later made to **not override** existing manual locks when turning on auto-block, that requires checking the owner **before `create_lock`** and skipping the create — a separate change.

### 3. Interaction with gaming mode

Gaming mode creates a 4h lock with owner `homeassistant-gaming-purple`.

**Without owner check:** If gaming mode is active and the auto-block timer expires, the `timer_finished` branch unconditionally releases the gaming lock. **BUG** — same class as C1.

**With owner check:** `timer_finished` branch checks owner == `homeassistant-auto-block`? No (it's `homeassistant-gaming-purple`) → skip release. Gaming lock survives. **Correct.**

The gaming mode auto-off automation (`office_light_gaming_mode_purple_auto_off.yaml`) already uses an owner check — it only releases when owner == `homeassistant-gaming-purple`. This confirms the owner-check pattern is the established solution in this codebase.

### 4. Double-release on timer expiry

The `timer_finished` branch releases the lock and turns off `input_boolean.auto_block`. Turning off the input_boolean re-triggers the automation (`auto_block_toggle`), firing the `off` branch, which releases again.

**Without owner check:** Second release hits an already-deleted lock → 404 error (harmless but noisy in logs).

**With owner check:** Second release checks owner. If sensor is stale (still shows `homeassistant-auto-block`), passes check → DELETE on deleted lock → 404. If sensor is fresh (shows `unlocked`, owner None), fails check → skips. Either way, harmless.

**Recommendation:** Add `continue_on-error: true` to the `release_lock` calls, or suppress the 404. Alternatively, restructure to avoid the double-release (e.g., don't turn off `input_boolean.auto_block` in the `timer_finished` branch — let the user see the timer expired and turn it off manually, or use a different mechanism). The double-release is a pre-existing issue, not introduced by the owner check.

---

## Alternative Solutions

### Alternative 1 — Server-side owner-aware DELETE (most robust)

**Change:** The locking service supports conditional DELETE: `DELETE /locks/{name}?owner={owner}` — only deletes if the current owner matches. Returns 409 Conflict if owner doesn't match.

**HA change:** Add a new rest_command `release_lock_if_owner`:
```yaml
release_lock_if_owner:
  url: "http://locking-service:3000/locks/{{ name }}?owner={{ owner }}"
  method: DELETE
```

The auto-block automation calls `release_lock_if_owner` with `owner: homeassistant-auto-block` instead of `release_lock`.

**Pros:**
- **Atomic** — the owner check and delete happen in a single server-side operation. No race condition with the 15s polling lag.
- No dependency on sensor freshness.
- Clean separation of concerns — the locking service enforces ownership.

**Cons:**
- Requires changes to the external locking service (not in this repo).
- All lock consumers would need to migrate to the owner-aware variant (or use both).
- More work to implement and deploy.

**Verdict:** Best long-term solution. Eliminates the race condition entirely. But requires cross-system coordination.

### Alternative 2 — Track own lock state via a helper (fragile)

**Change:** Add an `input_boolean.auto_block_owns_lock` helper. Set it to `on` when `create_lock` succeeds, set it to `off` when releasing. Check this helper before releasing instead of the sensor.

**Pros:**
- No dependency on sensor freshness — local state is instant.
- No external service changes needed.

**Cons:**
- **Can get out of sync** — if the lock expires naturally on the server (duration elapsed), the helper still says "owns lock". The next release would skip, leaving a stale helper.
- HA restart: `input_boolean` persists, but the lock state on the server may have changed during downtime. Helper says "owns" but server has no lock, or vice versa.
- Adds another piece of state to maintain.

**Verdict:** Fragile. Introduces a new sync problem. Not recommended.

### Alternative 3 — Owner check via sensor (recommended immediate fix)

**Change:** Add a template condition checking the lock owner before each `release_lock` call in the `timer_finished` and `off` branches. Remove the redundant `release_lock` from the `on` branch.

```yaml
# In timer_finished and off branches, before release_lock:
- condition: template
  value_template: >
    {{ is_state('sensor.office_light_lock_status', 'locked')
       and state_attr('sensor.office_light_lock_status', 'owner') == 'homeassistant-auto-block' }}
```

**Pros:**
- **Follows the established pattern** — `office_light_gaming_mode_purple_auto_off.yaml` already does this.
- Simple, minimal change — only touches `auto_block_control.yaml`.
- No external service changes.
- Fixes C1 for the `timer_finished` and `off` branches.
- **Bonus:** Fixes the broader HA-restart-wipes-all-locks bug (Scenario F3).

**Cons:**
- **Race condition** — 15s polling lag means the sensor may show stale owner data. Small window where the check can fail to protect a manual lock.
- Sensor unavailability causes release to be skipped (fail-safe, but the lock persists until natural expiry).

**Verdict:** Best immediate fix. Consistent with existing codebase patterns. The race condition is low-probability and can be eliminated later with Alternative 1 if it proves to be a real problem.

### Alternative 4 — Separate lock keys per consumer (architectural change)

**Change:** Instead of one lock per resource (`office_light`), each consumer uses its own lock key (`office_light_auto_block`, `office_light_manual`, `office_light_gaming`). The block-status binary sensor checks if **any** of these locks exist.

**Pros:**
- **No collision possible** — each consumer manages its own lock independently.
- No owner check needed — each consumer only touches its own lock.
- Eliminates the entire class of collision bugs.

**Cons:**
- Requires locking service to support multiple locks per resource (or the block sensor queries multiple keys).
- Changes to the block-status binary sensor (must check multiple lock keys).
- Bigger refactor — touches all lock consumers.
- The locking service's "one lock per key" model would need to change or be worked around.

**Verdict:** Most architecturally clean, but a large refactor. Overkill for fixing C1. Could be a future improvement if lock collisions become a recurring problem across multiple consumers.

---

## Recommendation

**Adopt Alternative 3 (owner check via sensor) as the immediate fix.**

### Rationale

1. **Consistency** — The gaming mode auto-off automation already uses this exact pattern. The fix brings auto-block control in line with established codebase conventions.
2. **Minimal scope** — Only `auto_block_control.yaml` changes. No external service modifications, no new helpers, no refactoring of other consumers.
3. **Fixes more than C1** — The owner check on the `off` branch also fixes the HA-restart-wipes-all-locks bug (Scenario F3), which is a broader issue affecting gaming mode and any other lock consumer.
4. **Acceptable trade-off** — The 15s polling race condition is low-probability and fail-safe (worst case: a manual lock is released prematurely, same as the current bug, but only in a narrow window). Alternative 1 can eliminate this later if needed.

### Specific changes to `auto_block_control.yaml`

1. **`timer_finished` branch:** Add owner-check condition before `release_lock`. Only release if owner == `homeassistant-auto-block`.
2. **`off` branch:** Add owner-check condition before `release_lock`. Only release if owner == `homeassistant-auto-block`.
3. **`on` branch:** Remove the pre-create `release_lock` (line 36) — it is redundant because `create_lock` (POST) replaces the existing lock.
4. **All `release_lock` calls:** Add `continue_on-error: true` to suppress 404 errors from the double-release on timer expiry.

### What this does NOT fix (deferred)

- **Scenario E / F2** (auto-block overriding manual locks): The `on` branch's `create_lock` POST replaces any existing lock. Preserving manual locks when turning on auto-block requires checking the owner before `create_lock` and skipping the create — a separate design decision about whether auto-block should override manual locks. Current behavior (override) appears intentional.
- **Timer reset on HA restart (F1):** The timer restarts from the selected duration, not the remaining time. This is a pre-existing issue unrelated to C1.
- **Race condition from 15s polling lag:** Eliminated by Alternative 1 (server-side owner-aware DELETE) if pursued later.

### Future improvement path

If the 15s polling race condition proves to be a real problem:
1. Implement Alternative 1 (server-side owner-aware DELETE) in the locking service.
2. Add `release_lock_if_owner` rest_command.
3. Migrate all lock consumers to the owner-aware variant.
4. The sensor-based owner check can then be removed (the server enforces ownership atomically).