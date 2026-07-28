---
title: "CI: Fail on Invalid HA Config & Write Back to PR Comment"
date: 2026-07-29
issue: 44
status: proposed
---

# Design — Improve CI to Fail on Invalid Config and Write Back to Comment

## 1. Problem Statement

The workflow at `.github/workflows/check-homeassistant-config.yaml` uses
`frenck/action-home-assistant@v1.4.1` to run
`python -m homeassistant --config . --script check_config` inside a Docker
container.

Two defects:

1. **CI never fails on invalid config.** The `check_config` script is
   well-known to **log errors via Python's `logging` module but exit with
   code 0** in most cases. The frenck action's final step is a plain
   `docker run` that streams stdout/stderr to the log and returns the
   container's exit code. Because `check_config` exits 0, the step is
   marked `success` even when the log is full of `ERROR:...` lines. The
   bundled `matcher.json` annotates the run log (rendering inline error
   annotations) but annotations do **not** fail the job.

2. **No PR feedback.** Errors are only visible buried in the run log.
   There is no comment posted to the PR summarizing what broke.

### Confirmed by inspecting the action source

The frenck `action.yaml` final step:

```yaml
- name: 🚀 Run Home Assistant Configuration Check
  shell: bash
  run: |
    docker run --rm \
      --entrypoint "" \
      -v $(pwd):/github/workspace \
      --workdir /github/workspace \
      "ghcr.io/home-assistant/home-assistant:${{ steps.version.outputs.version }}" \
        python -m homeassistant \
          --config "${{ steps.check.outputs.path }}" \
          --script check_config
```

No output capture, no exit-code inspection, no `grep` for error markers.
The matcher only affects log rendering. Hence the observed behavior.

## 2. Requirements

| # | Requirement |
|---|-------------|
| R1 | The CI job **must fail** (non-zero exit) when `check_config` reports errors. |
| R2 | On failure, error details are **posted as a comment** on the PR (no comment if check passes). |
| R3 | For `push` events (no PR), the job must still fail; commenting is skipped gracefully. |
| R4 | The existing setup (path rewrite, `data/` dir, `automations.yaml` touch, `ci-secrets.yaml`) must be preserved. |
| R5 | Pinned SHAs for third-party actions must be retained (supply-chain hygiene already in place). |
| R6 | Idempotent comments — re-running CI on the same PR always updates the same comment (edit, not create). |

## 3. Design Constraints & Facts

- `gh` CLI is preinstalled on `ubuntu-latest` runners and authenticated via
  the default `GITHUB_TOKEN`. To post PR comments the job needs
  `pull-requests: write` permission (or an explicit `permissions:` block).
- `check_config` error lines look like:
  ```
  ERROR:homeassistant.components.automation:Automation with alias 'Auto Block Control' could not be validated and has been disabled: extra keys not allowed @ data['actions'][0]['choose'][3]['default']. Got []
  ```
  Also relevant: `WARNING:...`, `Platform error ... - Integration '...' not found`,
  `Invalid config for ... (See: ..., line N)`.
- The `paths-ignore` block currently applies only to `pull_request`, not
  `push`. Editing `.github/**` on `main` therefore still triggers CI on
  push but is ignored on PRs — an asymmetry worth noting but out of scope
  to "fix" unless we choose to.
- The frenck action is a **composite** action; its steps run in our job's
  shell context. We cannot inject behavior between its internal steps, but
  we *can* run our own steps before/after it and read files it leaves
  behind (it leaves none — output goes only to stdout).

## 4. Approaches

Three approaches, ordered by how much of the frenck action we keep.

---

### Approach A — Wrap the frenck action: capture log via `tee`, grep for errors, comment

Keep `frenck/action-home-assistant` exactly as-is for running the check,
but **tee its output to a file**, then add two follow-up steps: one that
greps the captured log for error markers and fails the job, and one that
posts a PR comment.

**Mechanism:**

1. Replace the `uses:` step with a `run:` step that invokes the *same*
   docker command the action would run, but piped through `tee` so we
   keep a copy. — *OR* keep the `uses:` step and capture output by setting
   `continue-on-error: true` plus a wrapper. The clean variant is to stop
   using the action for the check step and inline the docker call (we
   still reuse the action's version-resolution + secrets-copy logic by
   keeping it for setup only). Simplest: drop the `uses:` entirely and
   inline.

   Actually the lowest-friction variant: keep `uses: frenck/...` but wrap
   it so its stdout is captured. Composite-action stdout cannot be
   captured directly by the caller in GitHub Actions (no step output
   forwarding for streamed logs). So we must either (a) inline the
   docker run ourselves, or (b) fork the action.

2. After the check, run:
   ```bash
   if grep -E '^(ERROR|Platform error|Invalid config for)' ha-check.log; then
     exit 1
   fi
   ```
3. Post comment via `gh pr comment ${{ github.event.pull_request.number }} --body-file ha-check.log` guarded by `if: github.event_name == 'pull_request'`.

**Pros:**
- Minimal change to existing mental model; frenck still handles version
  resolution, secrets, matcher registration.
- Reuses pinned, audited action for the heavy lifting.

**Cons:**
- **Cannot capture stdout of a composite `uses:` step from the caller.**
  GitHub Actions does not expose a composite action's internal step logs
  to downstream steps. `tee` only works if *we* own the `run:` block.
  This forces us to either inline the docker run (defeating "keep the
  action") or fork. So this approach collapses into B or C in practice.
- The matcher annotations would be lost if we inline, unless we re-add
  `::add-matcher::` ourselves.
- Duplicated logic: we'd re-implement version resolution / secrets copy
  if we inline, or maintain a fork if we don't.

**Verdict:** Conceptually clean but **technically blocked** by GitHub
Actions' lack of composite-action log forwarding. Not recommended as
stated; the inlined variant is Approach B.

---

### Approach B — Inline the docker check, capture output, fail + comment (replace frenck)

Drop `frenck/action-home-assistant` for the check step. Replicate its
essential behavior (version resolution via `.HA_VERSION` or `stable`,
`docker pull`, optional secrets copy, optional `--env-file`) in our own
`run:` steps, pipe `check_config` through `tee` to a log file, then grep
+ comment.

**Mechanism (sketch):**

```yaml
- name: Resolve HA version
  id: haversion
  run: |
    version="${{ inputs.version || '' }}"
    if [[ -z "$version" ]]; then
      version=$(<.HA_VERSION 2>/dev/null || echo stable)
    fi
    echo "version=$version" >> "$GITHUB_OUTPUT"
    docker pull -q "ghcr.io/home-assistant/home-assistant:$version"

- name: Stage fake secrets
  run: cp ci-secrets.yaml secrets.yaml

- name: Run check_config (capture output)
  run: |
    set +e
    docker run --rm --entrypoint "" \
      -v "$PWD":/github/workspace --workdir /github/workspace \
      "ghcr.io/home-assistant/home-assistant:${{ steps.haversion.outputs.version }}" \
      python -m homeassistant --config . --script check_config \
      2>&1 | tee ha-check.log
    # check_config exits 0 even on errors; ignore its code

- name: Fail on errors
  run: |
    if grep -E '^(ERROR|Platform error|Invalid config for)' ha-check.log; then
      echo "::error::Home Assistant configuration check failed — see comment / logs"
      exit 1
    fi

- name: Comment on PR
  if: github.event_name == 'pull_request' && failure()
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    PR_NUMBER: ${{ github.event.pull_request.number }}
  run: |
    # build a markdown body from the grep'd error lines
    {
      echo "## ❌ Home Assistant Config Check Failed"
      echo
      echo "<details><summary>Error output</summary>"
      echo
      echo '```'
      grep -E '^(ERROR|Platform error|Invalid config for)' ha-check.log
      echo '```'
      echo
      echo "</details>"
    } > comment.md
    existing=$(gh pr view "$PR_NUMBER" --json comments --jq \
      '.comments[] | select(.body | startswith("## ❌ Home Assistant Config Check Failed")) | .id' \
      | head -n1)
    if [[ -n "$existing" ]]; then
      gh pr comment "$PR_NUMBER" --edit "$existing" --body-file comment.md
    else
      gh pr comment "$PR_NUMBER" --body-file comment.md
    fi
```

With a top-level `permissions: { pull-requests: write, contents: read }`.

**Pros:**
- Full control over output capture and exit code — directly fixes R1.
- No dependency on frenck's internal step structure; survives upstream
  changes/breakage.
- We can re-register the matcher ourselves (`::add-matcher::`) to keep
  inline log annotations if desired.
- Comment step is trivial with `gh`.

**Cons:**
- We take ownership of version resolution + secrets staging; ~15 lines of
  bash we now maintain. Low complexity but non-zero.
- Loses the frenck action's future improvements (new HA channels, etc.)
  unless we manually bump.
- Without care, re-running CI posts a **new** comment each time (R6
  violation) — need comment-find-and-edit logic (see §5).

---

### Approach C — Fork/patch frenck action (or use a maintained alternative) + thin comment step

Fork `frenck/action-home-assistant` into this repo (or a sibling repo)
with a one-line patch to the final step:

```bash
docker run ... | tee /tmp/ha-check.log
grep -E '^(ERROR|Platform error|Invalid config for)' /tmp/ha-check.log && exit 1
```

…exposing the log path via an action output (`outputs: check-log`), then
consume it from our workflow with a separate comment step. Alternatively
adopt a community action that already fails + comments (none found that
both fail *and* comment; several only annotate).

**Pros:**
- Keeps the "one `uses:` line" ergonomics.
- Fork can be reused across other HA repos the user owns.
- Upstreamable as a PR to frenck (good citizenship).

**Cons:**
- Introduces a repo to maintain (or a vendored copy in `.github/actions/`).
- Slowest to set up; overkill for a single-repo config.
- Still need the comment step in the consumer workflow; doesn't save much
  over B.
- Fork drift risk if frenck releases fixes we don't pull.

## 5. Cross-Cutting Concerns

### 5.1 Idempotent PR comments (R6)

On re-runs and `synchronize` events we must not pile up duplicate
comments. Strategy: tag our comment with a hidden HTML anchor
`<!-- ha-config-check -->` and, before posting, search existing comments
for that anchor; if found, **edit** it (`gh pr comment --edit <id>`)
instead of creating a new one.

```bash
existing=$(gh pr view "$PR_NUMBER" --json comments --jq \
  '.comments[] | select(.body | contains("<!-- ha-config-check -->")) | .id' \
  | head -n1)
if [[ -n "$existing" ]]; then
  gh pr comment "$PR_NUMBER" --edit "$existing" --body-file comment.md
else
  gh pr comment "$PR_NUMBER" --body-file comment.md
fi
```

### 5.2 Push events (R3)

`github.event_name == 'push'` → no PR number. The comment step's `if:`
guard skips it. The "Fail on errors" step still runs and fails the job,
so pushes to `main` are protected. Optionally post to the commit's check
run via the API, but YAGNI — failing the check is sufficient for pushes.

### 5.3 Permissions

Add an explicit `permissions:` block (least privilege):

```yaml
permissions:
  contents: read
  pull-requests: write
```

The default `GITHUB_TOKEN` is otherwise read-only on `pull_request`
events for repos with restricted default tokens, which would make
`gh pr comment` fail silently.

### 5.4 Error-line detection robustness

`check_config` emits several distinct error shapes. The grep pattern set:

| Pattern | Meaning |
|---------|---------|
| `^ERROR:` | Python-logged errors (the example in the issue) |
| `^Platform error .* - Integration '.*' not found` | Missing integration |
| `^Invalid config for .* \(See .* line \d+\)` | Schema validation failure |
| `^WARNING:` | Warnings — **do not fail** (optional: include in comment) |

We fail only on the first three. Warnings are surfaced in the comment
body for visibility but never cause a red check, matching frenck's own
matcher severity mapping.

### 5.5 `paths-ignore` asymmetry

Currently `paths-ignore` is under `pull_request` only. A PR that edits
*only* `.github/workflows/check-homeassistant-config.yaml` will **not**
trigger the check, so a broken workflow file ships untested. Recommend
moving `paths-ignore` so it applies to both triggers, or removing it
entirely (the check is cheap). **Recommendation: remove `paths-ignore`
for `.github/**`** so workflow changes are themselves validated. Keep
ignoring `.taskfiles/**` and `docu/**` on both triggers. This is a small
bonus fix; call it out in the plan but keep it optional.

## 6. Recommendation

**Adopt Approach B** (inline the docker check, capture output, grep to
fail, `gh pr comment` with edit-if-exists logic), plus the §5
cross-cutting items.

Reasoning:

- Approach A is technically infeasible because GitHub Actions does not
  forward a composite action's internal step logs to the caller; we
  cannot `tee` output we don't own. The only way to "keep the action and
  capture output" is to fork it — which is Approach C.
- Approach C adds a maintenance surface (a fork/vendored action) for
  marginal ergonomic gain in a **single** repo. The frenck action's
  non-trivial logic is ~30 lines of bash; inlining it is cheaper than
  owning a fork and simpler than coordinating a cross-repo dependency.
- Approach B gives full control of the exit code (R1), makes the comment
  step trivial (R2), degrades cleanly on push (R3), preserves all
  existing setup steps (R4), and keeps pinned-SHA hygiene for
  `actions/checkout` (R5). The only owned code is ~25 lines of bash,
  which is easy to review and test.

### Concrete shape of the recommended workflow

```
checkout (pinned) → sed path → mkdir data → touch automations.yaml
→ resolve HA version + docker pull
→ stage ci-secrets.yaml as secrets.yaml
→ docker run check_config | tee ha-check.log   (continue-on-error: false, but ignore exit)
→ grep error markers → exit 1 on match         (R1)
→ gh pr comment always (edit-or-create, guarded to PR) (R2/R3/R6)
```

With `permissions: { contents: read, pull-requests: write }`.

## 7. Open Questions for the Implementer

1. Should warnings (`WARNING:`) also appear in the PR comment body even
   when the check passes? **Recommendation: yes, in a collapsed
   `<details>` block, only when present.**
2. Do we want to keep the frenck problem matcher for inline log
   annotations? **Recommendation: yes — re-register it via
   `::add-matcher::` pointing at a vendored copy in
   `.github/matchers/homeassistant.json` so the run log still gets
   annotated.**
3. Pin the HA docker image by digest in addition to tag? Trade-off:
   reproducibility vs. needing manual bumps to catch new config errors
   introduced by HA releases. **Recommendation: keep tag-based (`stable`)
   for the check; the whole point is to catch breakage against current
   HA.**

## 8. Out of Scope

- Fixing the actual HA config errors (separate issues).
- Matrix-testing against `beta`/`dev` (possible future enhancement; the
  frenck README shows the pattern).
- Posting a comment on push events via check-run API (YAGNI).