# Changelog

Dated record of Hermes Agent upgrades applied to a single production
host. See [`README.md`](./README.md) for format and privacy policy.

---

### 2026-09-28 09:15 — Release Watcher: check_failed mystery solved — install format migrated, v0.21.4/v0.21.5 shipped, code now runs 4,108 canary commits ahead of v0.21.5

**Status:** Monitoring + diagnosis. No upgrade performed by the watcher (one
was executed upstream of it — see "Daily updates" below).

**Why the watcher fired:** monitor output flipped from
`latest=v0.21.3 installed=v0.21.3 behind=0` to
`check_failed installed=unknown latest=v2026.9.24`.

**What actually happened (verified):**

- **Releases:** v0.21.4 (v2026.9.21, 2026-09-21) and v0.21.5 (v2026.9.24,
  2026-09-24) shipped since v0.21.3. Both are "rolling-up" patch tags for
  downstream consumers; curated notes are deferred to v0.22.0. v0.21.4
  window: ~1,812 PRs. v0.21.5 window: ~460 PRs. (Notes fetched this run.)
- **Install format migrated 2026-09-25/26:** the update pipeline moved to a
  PM-managed ("bootstrap") layout — `/root/.hermes/installs/76f6e7e5145b5f39/`
  (env venv, tools store `/root/.hermes/tools/`, ffmpeg 9.0.1 present).
  `which hermes` now resolves to the env venv, whose workspace copy reports
  `Hermes Agent vunknown` (no stamp/no git) — the direct cause of
  `installed=unknown`. The real, blessed checkout remains
  `/usr/local/lib/hermes-agent` at `v0.21.5+4108.g105568c.dirty`
  (upstream `105568c1`) and the gateway runs from it (systemd unit
  ExecStart, PID 1260418, started 2026-09-26 16:32 EDT).
- **Head not behind — ahead:** HEAD is 4,108 commits AHEAD of the v0.21.5
  tag (canary-style daily fast-forwards continue; reflog: fast-forwards
  9/22, 9/23 ×2, 9/25 ×2, 9/26, 9/28 09:00). Driver still unidentified —
  no system timer, no user timer, no crontab entry runs updates. It
  predates this run and needs Matt's eyes (candidate: gateway housekeeping
  or a desktop-app-triggered sync).
- **`hermes update` CLI now warns** "pulled new code but did not restart
  running gateways (mixed sys.modules)" — cosmetic-but-real: the gateway
  has been running since 9/26 on the e295174 snapshot; today's 09:00
  fast-forward to 105568c is live on disk, not in the gateway's memory.
  Restarting is the fix; requires Matt's explicit go-ahead.
- **Post-update state:** `active_profile` had reset to default; restored
  to daily via `hermes profile use daily` this run (verified). Config
  format migrated v46 → v48 across all 6 profiles during the 9/28 update
  (no new settings required). Cron: single-store discipline intact —
  global store has 23 jobs (21 active / 2 paused), matching the pre-9/28
  baseline; ticker heartbeat fresh (59s); scheduler host = system gateway
  (PID 1260418). No daily-mirror double-dispatch observed (both 9/28 09:00
  executions sourced `builtin`).
- **Local patch still intact:** `.hermes`-basename profile fallback in
  `hermes_cli/profiles.py` (`normalize_profile_name`) survives — confirmed
  present post-fast-forwards (uncommitted, git status shows
  `M hermes_cli/profiles.py`).
- **Unapplied local patch surfaced (stash@{0}, 2026-09-04):** a dashboard
  `gated_auth_middleware` fallback accepting the desktop's
  `X-Hermes-Session-Token` header (fixes Electron-mediated file-download
  401s via saveGatewayFile → /api/fs/download). `X-Hermes-Session-Token`
  still exists upstream (`web_server.py:358`) but the middleware has no
  such fallback on current HEAD. Unreviewed upstream-side fix may exist;
  needs Matt's decision: apply, or drop. A second old autostash
  (stash@{1}, 2026-06-22) shows empty output to `git stash show`.
- **Watcher script fixed:** `/root/.hermes/scripts/hermes-release-check.sh`
  (and its daily-profile mirror) now (a) probes the blessed checkout venv
  first and the PM env venv second instead of trusting PATH, (b) computes
  `behind` from the checkout's git (the new version line never prints it),
  (c) reports the stable base semver (v0.21.5) instead of the growing
  canary suffix so weekly syncs don't re-fire it as noise. Post-fix output:
  `latest=v0.21.5 installed=v0.21.5 behind=0` (behind grew to 17 mid-run —
  all desktop/UX/gateway-restart fix commits; ~1,000 commits/day cadence).
- **Known-warning triage:** doctor's "install out of sync (ffmpeg)" is a
  false alarm — the ffmpeg 9.0.1 artifact lives in `/root/.hermes/tools/`
  (PM-managed) while the doctor check still scans the legacy layout;
  `ffmpeg` resolves in PATH and video_gen deps passed. doctor also wants
  the dashboard unit to carry `RestartPreventExitStatus=78` (exit-78
  permanent refusals currently crash-loop it) — one-line unit patch, next
  time a dashboard restart is approved. Two pre-existing doctor warnings
  unchanged (spotify/tts system deps; x_search missing XAI_API_KEY).

**Relevant highlights from the v0.21.4/v0.21.5 windows** (headless-VPS
lens): `--format stream-json` CLI output; `skills.auto_load` pinning;
gateway `decline` unauthorized-DM behavior; `mcp.discovery_concurrency`
cap; `session_search` after/before bounds + OR-relaxed retry; `hermes
sessions set-journal-mode`; host-wide gateway singleton lock + Desktop
attaching to the running backend; per-profile stop/start/restart under
the host multiplexer + `gateway.standalone` opt-out; webhook deliveries
mirrored into the target chat session; kanban design pass (two-column
ticket modal, markdown task text) plus a run of kanban fixes in the
canary window (aux-LLM key resolution, per-task skip events, PR
acceptance as the assignee profile, blocked-task surfacing); cron
multiplex fixes (satellite routing, routed manual runs, liveness naming
the ticking scheduler, DST-proof auto-resume); `attachments.storage`
config; `/goal` + queued prompts in the live dock; GPT-6 Sol/Terra/Luna
+ Claude Opus 5.5 catalogs; hot-path perf work (config loading, tool
registry, gateway message handling). Skipped as not applicable:
Desktop/TUI-only UX, voice, catalog plugins for platforms not in use.

**Breakage:** none observed. Dashboard reachable (flags intact incl.
`--open-profile daily`), Honcho connected, all 6 profiles healthy.

**Action queue for Matt (all require his go-ahead):**
1. Decide the daily-update driver (unknown since 9/15) — want it, or want
   it found and stopped?
2. A gateway + dashboard restart to load the 9/28 code (or fold into the
   next deliberate update).
3. `hermes gateway migrate --multiplex` — update output reports the
   profile gateways are still separate; multiplex folds them under the
   host gateway (aligns with per-profile start/stop in v0.21.5).
4. Stash@{0} middleware patch: apply or drop.
5. Dashboard unit: add `RestartPreventExitStatus=78`.
6. Old autostash stash@{1} (6/22): `git stash show -p` review, then drop.

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

---### 2026-09-14 — Release Watcher false positive: date-tag vs semver comparison

**Status:** No upgrade performed. Monitoring fix only.

**Event:** The `hermes-release-check.sh` monitor fired reporting
`latest=v2026.9.11 installed=v0.21.2 behind=813`. Investigation showed
v2026.9.11 and v0.21.2 are the **same release** (GitHub title:
"Hermes Agent v0.21.2 (v2026.9.11)", published 2026-09-11) — Hermes ships
each release under both a semver tag and a date tag. The script compared
the date tag against `hermes --version`'s semver, which would have
false-positived every weekly run.

**Fix:** Script now resolves the release title and, when it contains the
installed version string, reports the installed form as `latest`. Patched
both copies (`/root/.hermes/scripts/` and
`/root/.hermes/profiles/daily/scripts/`); verified both now output
`latest=v0.21.2 installed=v0.21.2 behind=813`.

**Also noted:** HEAD of the installed repo is `v2026.9.11-2-g31d0a2428e`
(two local commits after the release tag); gateway was restarted onto this
code 2026-09-11 16:41 EDT. The `behind=813` figure is hermes's cached count
of *unreleased* `main` commits (fresh fetch shows 937 on 2026-09-14) —
normal drift, not a new release.

---

### 2026-09-21 — Post-upgrade verification: v0.21.3 (v2026.9.14) confirmed installed and healthy

**Status:** Verification record. The upgrade itself was performed outside
any recorded session (no changelog entry was made at the time); this entry
records the verified current state instead of a fabricated procedure.

**Release:** v0.21.3 (v2026.9.14), published 2026-09-14 16:04 UTC. Patch
release rolling up ~338 PRs since v0.21.2, cut primarily to ship the
remote-gateway refresh-token coalescing fix (Desktop sessions revoked on
refresh bursts) and the state.db duplicate-writer-handle fix to Cloud/
release-tag consumers. Full curated notes for the window deferred to
v0.22.0 upstream.

**When it went live:** Repo fast-forwarded daily since 2026-09-15 10:00 EDT
(git reflog: daily `merge origin/main` fast-forwards 9/15 → 9/21, each
followed by gateway + dashboard restarts at varying times — driver not
identified; no cron job or system crontab entry performs updates). v0.21.3
code has therefore been live since on or before 2026-09-15.

**Verified 2026-09-21 ~09:00 EDT:**
- `hermes --version` → v0.21.3 (2026.9.14), upstream ea0c2b82; `Up to date`
- HEAD `ea0c2b820b` (2026-09-21), 0 commits behind origin/main; the
  v0.21.3..HEAD delta (≈4,926 non-merge commits) is already installed and
  running — the next release will be a restart-only event for this host
- `active_profile` had been reset to `default` by the update(s); restored
  to `daily` via `hermes profile use daily` this run; file verified
- Local `.hermes`-basename profile patch (`normalize_profile_name()`,
  hermes_cli/profiles.py ~line 183) — still present after the
  fast-forwards (upstream has not shipped its own fix)
- Rogue `hermes-gateway-daily.service` user unit: still disabled + dead;
  no ticker heartbeat revival
- Venv: openpyxl 3.1.5 / pdfplumber 0.11.10 / httpx 0.28.1 present;
  budget-app service active
- `hermes doctor`: no advisories, all DBs healthy in WAL mode,
  version files consistent (0.21.3)
- Cron: single-store discipline intact (0 job-ID overlap between global
  store and daily mirror fire locks)

**Breakage:** None observed.

---

### 2026-09-21 09:08 — Release Watcher: behind=0→2 race, desktop-only commits

**Status:** Monitoring note. No upgrade performed.

**Event:** The weekly watcher fired on `behind=0→2` at 09:01 EDT. Cause: a
race, not a release — upstream pushed 2 commits (`00e08cec30` fix(desktop)
Linux SIGTRAP crash diagnostics, `274bc7b8f6` docs(desktop)) at 09:00 EDT,
one minute before the run, right after the morning fast-forward + gateway
restart (08:53/08:55 EDT). `latest` is still v0.21.3 (matching installed);
the v0.21.3 tag has no GitHub release object (the release lives on the
date tag `v2026.9.14`, target `main`).

**Verified this run:** read-only fetch moved origin/main to `274bc7b8f6`
(HEAD stays at `ea0c2b82` until the next daily sync); both pending commits
are desktop/Electron-only — zero impact on this headless VPS. Re-checked
post-update state alongside the entry above: active_profile=daily,
dashboard `--open-profile daily` intact, `normalize_profile_name()` patch
intact, rogue `hermes-gateway-daily.service` absent, ticker heartbeat
fresh, `hermes -p default cron status` reports the host gateway (PID
2222314) as scheduler host under the new host-topology reporting.

---
