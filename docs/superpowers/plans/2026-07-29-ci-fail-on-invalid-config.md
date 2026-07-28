# CI: Fail on Invalid HA Config & Comment on PR — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the `frenck/action-home-assistant` composite action with an inline docker-based HA config check that fails the workflow on errors and posts a PR comment with the error details.

**Architecture:** Inline the ~30 lines of bash from the frenck action (version resolution, docker pull, secrets staging, docker run), pipe `check_config` output through `tee`, grep for error markers to determine pass/fail, and use `gh pr comment` with edit-if-exists logic to post failure details.

**Tech Stack:** GitHub Actions, bash, Docker (`ghcr.io/home-assistant/home-assistant`), `gh` CLI

---

### Task 1: Create feature branch and write the updated workflow

**Files:**
- Create: `feature/ci-fail-on-invalid-config` branch
- Modify: `.github/workflows/check-homeassistant-config.yaml` (replace frenck action with inline steps)
- Optional Create: `.github/matchers/homeassistant.json` (problem matcher for inline log annotations)

- [ ] **Step 1: Create the feature branch**

Run: `git checkout -b feature/ci-fail-on-invalid-config`

- [ ] **Step 2: Replace the workflow file with the inline implementation**

Replace the current `.github/workflows/check-homeassistant-config.yaml` with:

```yaml
---
name: Check Home Assistant Configuration

on:
  push:
    branches:
      - main
  pull_request:
    types:
      - opened
      - synchronize
    paths-ignore:
      - .taskfiles/**
      - docu/**

permissions:
  contents: read
  pull-requests: write

jobs:
  home-assistant:
    name: Home Assistant Core Configuration Check
    runs-on: ubuntu-latest
    steps:
      - name: ⤵️ Check out configuration from GitHub
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false

      - name: Update external directory path for CI
        run: |
          sed -i 's|- /data|- ./data|g' configuration.yaml

      - name: Create data directory
        run: mkdir -p ./data

      - name: Create files
        run: touch automations.yaml

      - name: Resolve Home Assistant version
        id: haversion
        run: |
          version="stable"
          if [[ -f "./.HA_VERSION" ]]; then
            version=$(<"./.HA_VERSION")
          fi
          echo "version=${version}" >> "$GITHUB_OUTPUT"
          docker pull -q "ghcr.io/home-assistant/home-assistant:${version}"

      - name: Stage CI secrets
        run: |
          if [[ -f "ci-secrets.yaml" ]]; then
            cp ci-secrets.yaml ./secrets.yaml
          fi

      - name: 🚀 Run Home Assistant Configuration Check
        id: check
        run: |
          set +e
          docker run --rm \
            --entrypoint "" \
            -v "$(pwd):/github/workspace" \
            --workdir /github/workspace \
            "ghcr.io/home-assistant/home-assistant:${{ steps.haversion.outputs.version }}" \
            python -m homeassistant --config "." --script check_config \
            2>&1 | tee ha-check.log
          grep -qE '^(ERROR|Platform error|Invalid config for)' ha-check.log && echo "status=failure" >> "$GITHUB_OUTPUT" || echo "status=success" >> "$GITHUB_OUTPUT"

      - name: ❌ Fail on configuration errors
        if: steps.check.outputs.status == 'failure'
        run: |
          echo "::error::Home Assistant configuration check found errors"
          grep -E '^(ERROR|Platform error|Invalid config for)' ha-check.log
          exit 1

      - name: 💬 Comment failure on PR
        if: github.event_name == 'pull_request' && steps.check.outputs.status == 'failure'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
        run: |
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

Key changes from current workflow:
- Removed `paths-ignore` for `.github/**` so workflow changes get validated
- Added `permissions:` block for `pull-requests: write`
- Replaced `uses: frenck/action-home-assistant` with inline steps:
  - `Resolve Home Assistant version` — pulls the docker image
  - `Stage CI secrets` — copies `ci-secrets.yaml` to `secrets.yaml` if present
  - `Run Home Assistant Configuration Check` — runs check_config, pipes to `tee ha-check.log`, stores status
  - `Fail on configuration errors` — exits 1 if status is failure
  - `Comment failure on PR` — posts/updates a PR comment on failure, guarded to PR events

- [ ] **Step 3: Commit and push the feature branch**

```bash
git add .github/workflows/check-homeassistant-config.yaml
git commit -m "ci: fail on invalid HA config and post PR comment

Replace frenck/action-home-assistant with inline docker-based HA config
check that properly fails the workflow when configuration errors are found
and posts a PR comment with error details.

Closes #44"
git push -u origin feature/ci-fail-on-invalid-config
```
