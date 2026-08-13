# Changelog

Dated record of Hermes Agent upgrades applied to a single production
host. See [`README.md`](./README.md) for format and privacy policy.

---

### 2026-08-13 — Backfill entry: v0.18.2 → v0.20.0 upgrade history

**Status:** Retrospective reconstruction. The entries below were backfilled
on August 13, 2026, after discovering that four upgrades had been performed
without any changelog entries being recorded. Dates are approximate where
exact session records were unavailable (the state.db was locked during
reconstruction). Future upgrades will be logged in real time per the
post-update feature review procedure added to the `hermes-agent` skill.

#### v0.18.2 → v0.19.0 (~July 20–23, 2026)

**Release:** v0.19.0 (v2026.7.20) — "The Quicksilver Release"

**Context:** A Kanban card was created on July 16 titled "Upgrade Hermes
Agent v0.18.2 to latest - 247 commits behind." By July 23, a session
confirmed the upgrade was already complete ("Hermes was already upgraded
to v0.19.0. Task is outdated"). The exact upgrade date and procedure were
not recorded. The card was archived as stale.

**Known effects on this deployment:**
- The Projects feature (`hermes project`) was introduced in this release
  window (committed June 25, shipped in v0.19.0). Seven projects were
  created but never bound to Kanban boards.
- Board-to-project binding (`hermes project bind-board`) became available.
- The `--project` and `--workspace worktree` flags on `hermes kanban create`
  were available but never used on any card.
- The `kanban-orchestrator` skill (v3.0.0) did not document these features.

**Skills that should have been patched but weren't:**
- `kanban-orchestrator` — should have added `--project` + `--workspace worktree`
  guidance (patched August 13, 2026 as v3.1.0)
- `hermes-agent` — should have added a post-update feature review procedure
  (patched August 13, 2026)

**Breakage:** None recorded.

#### v0.19.0 → v0.19.1 (~July 30, 2026)

**Release:** v0.19.1 (v2026.7.30)

**Context:** Patch release. No session record of the upgrade was found.
Inferred from the August 6 session which references "Hermes updated from
v0.19.1 to v0.20.0" — meaning v0.19.1 was the prior version.

**Known effects on this deployment:**
- Bug fixes only. No new features relevant to this deployment identified.

**Breakage:** None recorded.

#### v0.19.1 → v0.20.0 (August 6, 2026)

**Release:** v0.20.0 (v2026.8.3) — "The Herald Release"

**Context:** Upgraded on August 6, 2026. This is the only upgrade with a
detailed session record (`@session:daily/20260806_152949_91ec9e`).

**Known effects on this deployment:**
- **Budget app crash (103,999 restart loops):** The update regenerated the
  shared Python venv from Hermes's `pyproject.toml`, wiping three packages
  the budget app depends on (`openpyxl`, `pdfplumber`, `httpx`). Uvicorn
  crashed on import, systemd restarted it, repeat.
  - **Fix:** Reinstalled missing packages, restarted service.
  - **Prevention:** Added `ExecStartPre=/root/budget-app/start.sh` — checks
    imports and auto-reinstalls from `requirements.txt` if Hermes wiped them.
  - **Documented in:** `hermes-environment-setup` skill →
    `references/post-update-venv-package-loss.md`
- **Daily vocab email missed at 3:30 PM:** The cron scheduler inside the
  dashboard process missed the tick, likely due to a dashboard restart
  earlier that morning resetting the cron ticker's internal state.
  - **Fix:** Manually triggered the job at 6:58 PM.
  - **Root cause:** Known cron ticker stall issue (documented in AGENTS.md).

**Major features in this release relevant to this deployment:**
- Grounded citations skill (research with verifiable sources)
- Outbound webhooks (signed lifecycle events to HTTP endpoints)
- CLI power commands (`!` shell mode, `/init`, `/diff`, `/context`, `/focus`)
- Redirect-mid-turn (course-correct without `/stop`)
- Self-healing tools (truncated output spills to file, patch detects
  already-applied edits, searches recover from no-match)
- Compression overhaul (proactive pruning, micro-compaction, guaranteed
  N-message tail, progress-aware timeouts)
- Smart approvals (`hermes approvals suggest`, consecutive-denial breaker)
- Tool-calling iteration limit raised 90 → 500

**Features NOT relevant to this headless VPS deployment:**
- Voice/TTS (streaming conversational voice, barge-in, wake words)
- Desktop app artifacts, plugin SDK, quick-entry
- A2A protocol (no other agents to talk to)
- Buzz/Nostr platform
- Various gateway platform parity features

**Skills that should have been patched but weren't:**
- `hermes-agent` — should have added post-update feature review procedure
  (patched August 13, 2026)
- `kanban-orchestrator` — should have documented `--project` + worktree
  (patched August 13, 2026 as v3.1.0)

**What was actually done right:**
- Venv package loss was diagnosed and fixed quickly
- Self-healing wrapper was added to prevent recurrence
- Both issues were documented in the `hermes-environment-setup` skill
- Memory entries were saved for both issues

---

### 2026-06-08 — v0.16.0 → v0.16.0+4 (pre-upgrade plan)

**Status:** Pre-upgrade analysis. The upgrade itself has not been run
yet; this entry exists so that the post-upgrade entry can be a short
verification record, not a fabrication.

**Upstream state:** 4 commits behind `NousResearch/hermes-agent` `main`.
3 distinct PRs.

**Scope (per `git log HEAD..@{u} --stat`):**

| File | +/- | Notes |
|---|---|---|
| `apps/desktop/electron/main.cjs` | +25 / -0 | Desktop app only |
| `plugins/observability/nemo_relay/README.md` | +13 / -17 | Plugin docs |
| `plugins/observability/nemo_relay/__init__.py` | +9 / -2 | Plugin code |
| `tests/plugins/test_nemo_relay_plugin.py` | +71 / -9 | Plugin tests |

Total: 4 files, +134 / -26 lines.

**PR-by-PR breakdown:**

- **`fix(desktop): stream the transcript while the window is backgrounded`
  (#42399)** — Touches `apps/desktop/electron/main.cjs`. Affects only
  the Electron desktop application. **Not relevant** to a headless
  VPS / TUI / gateway deployment.

- **`fix(nemo-relay): align adaptive config with tool_parallelism mode`**
  — Touches the optional `nemo-relay` observability plugin's
  adaptive configuration and its alignment with `tool_parallelism`.
  Relevant only if that plugin is enabled on the host; the upgrade
  is otherwise a no-op for plugin-disabled deployments.

- **Merge commits** — Two `Merge` commits (`#42331` and its parent)
  that bring the above fix into `main`. No code change of their own.

**Pre-upgrade analysis (what we expect):**

- No changes to the core agent loop, tool runtime, skills, gateway,
  or configuration schema in this patch set.
- No security or authentication-related changes.
- No dependency manifest changes (`pyproject.toml`, `requirements.txt`,
  etc. are untouched in the diff).
- For a headless deployment that does not use the `nemo-relay`
  observability plugin, the upgrade is functionally a no-op aside
  from a clean `git` history on the installed code.

**Procedure to be followed (post-upgrade entry will record actual results):**

1. Snapshot the installed package directory.
2. Run the upstream-supplied update command.
3. Verify the post-upgrade version string.
4. Smoke test: confirm the agent starts, the gateway is reachable,
   and one trivial tool call succeeds.
5. Update this changelog with the verified post-upgrade state.
6. Commit and push.

**Decision points (deferred to the post-upgrade entry):**

- Whether the nemo-relay observability plugin is actually in use on
  this host (relevant to whether the one code-bearing PR has any
  observable effect).
- Whether to keep the snapshot retained or rotate it after a clean
  smoke test.

---