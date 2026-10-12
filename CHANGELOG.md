# Changelog

Dated record of Hermes Agent upgrades applied to a single production
host. See [`README.md`](./README.md) for format and privacy policy.

---

### 2026-10-12 09:20 — Release Watcher: v0.21.6 + v0.21.7 shipped (Oct 8 / Oct 12), monitor line garbled by dead gh token + update race, patches re-applied

**Status:** Monitoring. No manual upgrade performed; the Oct 8 v0.21.6 update
(desktop app) was external and is recorded here retroactively.

**Releases:**
- **v0.21.6** — published 2026-10-08 07:51 EDT. First release cut by the new
  stable release pipeline. Rolls up 8,867 non-merge commits since v0.21.5
  (2,106 PRs, 3,027 closed issues). Notable: dashboard auth hardening
  (4 Tenable findings — X-Forwarded-For rate-limit reset, unbounded audit-log
  writes, no body-size limit on /auth/, non-loopback login-code redirect),
  git filter hardening against untrusted repos' clean/smudge/process filters,
  email gateway display-name allowlist fix, unified Kanban workflow,
  summary-first `hermes status`, cron-store writability warnings, persistent
  shell fixes. Was **already installed Oct 8** (desktop app; gateway PID
  985051 + dashboard 986160 restarted 09:11–09:13 EDT that day).
- **v0.21.7** — published 2026-10-12 01:53 EDT. Rolls up 740 non-merge
  commits / 204 PRs since v0.21.6. Notable: **source installs now follow
  tagged releases by default** (`hermes update --set-channel main` to keep
  following every commit; channel switch also in desktop About ▸ Updates),
  reasoning survives `chat --resume` and gateway turns, update fetches use
  the GitHub token when set and report rate limits honestly.

**Why the monitor fired garbled (`latest=` empty, behind=791):** three
overlapping causes, all verified live:
1. **gh token for BenkoMatt is dead** (401 Bad credentials via
   /root/.config/gh/hosts.yml; the `GITHUB_TOKEN` PAT in /root/.hermes/.env
   is also 401). `gh release list` printed nothing → empty tag field.
   Impact: github skill API ops, BenkoMatt-private git-over-HTTPS pushes
   (Update Record/JLP use SSH and are unaffected — verified).
2. **Update race at 09:00 EDT:** the desktop app drove a `hermes update`
   while the monitor ran — update.lock stamped 09:00, autostash 09:01,
   fast-forward to 3f2abce7f4 09:04, mid-pull the monitor saw origin/main
   791 ahead → the transient `behind=791` line. Both numbers were noise;
   settled state: HEAD = 3f2abce7f4 = v0.21.7+144.g3f2abce.dirty, **behind
   origin/main = 0**.
3. `xargs` on gh's empty output returns "" not "unknown", so the 401 path
   printed a blank tag instead of check_failed (monitor hardening queued).

**Patches re-applied cleanly after the pull (verified 09:07):** both
`.hermes`-profile fixes are present in the working tree again —
profiles.py basename→default mapping (7 lines) and the WS loopback-origin
carve-out in _ws_host_origin_reason (15 lines, re-derived shape matching the
stash). stash@{0} (hermes-update-autostash-20261012-130108) is the sync's
own copy — redundant with the working tree, safe to drop (needs Matt).
Stash@{1..3} unreviewed. This is the first sync where autostash re-apply
worked without manual re-application.

**Restart debt (desktop-owned processes; needs Matt):** gateway PID 985051
started Oct 8 09:11 still serves the 25a71a74 snapshot — three update drops
since (8a33891, 25a71a7→10/8 was already in, 3f2abce) — today's pull of
`~144 commits to v0.21.7+` lives on disk, not in gateway memory. Dashboard
PID 986160 (Oct 8, desktop-launched) shows the "manual restart still
pending" banner for today's drop. Cron unaffected: ticker heartbeat fresh,
host gateway hosts all 7 profiles, 21 active jobs.

**Decisions needed from Matt:**
1. **Update channel:** stay on the new default (stable/v0.21.7 tags — stops
   the weekly behind-N churn; next roll-up ~144 commits pending as
   v0.21.8) or `hermes update --set-channel main` (current behavior).
2. **Re-auth gh** (`gh auth login`, interactive; Matt-side) + refresh
   `GITHUB_TOKEN` in /root/.hermes/.env, or all latest-release lookups and
   private-HTTPS pushes stay broken.
3. Dashboard relaunch to load today's code (desktop-side action).

**Skills:** 4 user-modified bundled skills (github, hermes-agent, himalaya,
pdf). No new drift this week. hermes-agent stale-copy merge into stock
v3.2.0 still pending (carried from 9/28). himalaya 10/5 fix intact.

*(Committed together with the 10/5 entry that was left uncommitted.)*

---

### 2026-10-05 09:15 — Release Watcher: quiet week (v0.21.5, behind=2), overnight autostash dropped the desktop-session patch; re-applied + himalaya drift re-fixed

**Status:** Monitoring + patch restoration. No upgrade performed.

**Why the watcher fired:** monitor flipped from last week's
`check_failed installed=unknown latest=v2026.9.24` to
`latest=v0.21.5 installed=v0.21.5 behind=0` — the 9/28 script fix confirmed
working. During this run HEAD fast-forwarded to `71574220`, 2 upstream
commits landed behind (plugins entry-point `hermes status` crash fix +
its test), and the monitor line self-corrected to `behind=0` after the
refetch. **Installed (gateway memory) vs code on disk:** gateway PID
1553935 started **2026-10-05 08:02 EDT** — the overnight/weekly 09:00
refresh finally restarted it, so gateway memory and disk now match
(71574220 = v0.21.5+7249). The "gateway restart debt" from 9/28 cleared
itself. No restart performed or needed this run.

**Found + fixed this run:**

1. **Overnight autostash dropped the desktop-session patch.** The
   (still-unidentified) daily sync autostashed local mods at 01:33 and
   left them stashed instead of re-applying → working tree clean but
   BOTH `.hermes`-profile fixes missing:
   - `normalize_profile_name()` `.hermes`→`default` basename mapping
     (AGENTS.md re-check requirement; upstream still lacks any native
     fix — grep of profiles.py/_install_repair/_launchers confirms).
   - WS loopback-origin allowance in `web_server_chat.py` (desktop
     renderer's `http://127.0.0.1:<port>` origin vs remote dashboard).
   **Action taken:** profiles.py hunk re-applied from stash@{0} verbatim
   (`git apply --check` passed, 7 insertions). The WS hunk did NOT
   apply — upstream 2026-10-03 `1cb25c3e33` refactored
   `_ws_host_origin_reason` (single `origin_mismatch` return, new
   `parsed is None` guard). Re-implemented the loopback-origin branch
   on the new shape: loopback origins return None (allowed) before the
   origin_mismatch return, only for http/https origins with a hostname,
   preserving the fail-closed path for malformed authorities. Compile
   check + repo tests pass.
   **Net risk:** none beyond the historical one — HEAD is v0.21.5+7249;
   the patch is current again. But the sync's autostash re-apply step is
   evidently broken (2nd occurrence after 9/28); every future sync can
   silently strand these fixes again. **Needs Matt:** identify the sync
   driver (was on the 9/28 action queue, still unknown) or accept a
   watcher-side post-sync patch-recheck.

2. **Himalaya skill re-drifted after the 9/28 restore.** Our copy again
   reversed `himalaya message move/copy` args (`move 42 "Archive"` vs
   CLI's `<TARGET> <ID>...`) plus a wrong `--dir` flag on attachment
   download (actual: `--downloads-dir`). `hermes skills reset himalaya
   --restore` prompted for confirmation and cancelled non-interactively;
   fixed manually with sed on the global copy + all 6 profile mirrors
   (verified: no reversed examples remain, `--downloads-dir` correct,
   mirrors byte-identical to the global fixed copy). Kept a
   timestamped `.bak-watcher-20261005` next to the file. Remaining diff
   vs stock is prose only ("target folder comes first" annotations).
   **Note himalaya skill is now effectively correct-but-user-modified;
   future drizzle from upstream will re-drift; the 9/28 root cause
   (upstream briefly shipped the wrong order?) unknown — treat the CLI
   --help as ground truth.**

3. **hermes-agent skill: stale copy retained deliberately.** Drift grew
   to 38 files (stock is now 1223-line v3.2.0 SKILL.md + new reference
   refs incl. `project-context-files.md`, `native-mcp.md`,
   `security-privacy.md`, `slash-commands.md`). Ours is v2.1.0 + 14
   Matt-specific reference docs that `--restore` would delete. NOT
   auto-restored. Needs a deliberate merge session: fold our 14
   reference docs into the stock v3.2.0 layout, then re-baseline.

**Releases:** none since v0.21.5 (v2026.9.24); v0.21.4/v0.21.5 were
rolling patch tags with curated notes deferred to v0.22.0.

**Relevant in the 2-behind commits (both now counted ahead after refetch):**
- `e473f5a9c9` fix(plugins): skip default-on checks for entry-point paths
  → fixes `hermes status` crashing on Windows when one pip-installed
  plugin's spec (module:attr) is misread as a plugin directory and
  hiding all other plugins on profiles that hadn't enabled it. Linux-
  safe (colon paths are legal), cosmetic for us unless entry-point
  plugins appear. No action.
- `9192728734` test(plugins): entry-point spec must not abort plugin
  status. No action.

**Unchanged action queue (carry from 9/28; still needs Matt):**
1. Identify the daily-update driver (unknown since 9/15) AND its broken
   autostash-reapply — wants a decision: find/stop/fix, or have the
   watcher re-apply patches post-sync each run.
2. `hermes gateway migrate --multiplex` (folds 6 profile gateways under
   the host multiplexer; aligns with per-profile start/stop in v0.21.5)
   — note `hermes -p default cron status` still shows host gateway
   serving ALL 7 profiles already (default, daily, research, reviewer,
   security, webdev, work); verify the migrate output actually changes
   anything before running it.
3. Stash@{0} (now empty of profiles.py hunk; WS loopback fix re-derived,
   not identical) — recommend dropping stash@{0} once the re-applied
   working-tree state is confirmed good (it is this run); stash@{1}
   (9/4 dashboard middleware session-token fallback) + stash@{2} (6/22
   empty) still unreviewed, still needs Matt.
4. Dashboard unit: add `RestartPreventExitStatus=78` (9/28 finding,
   unchanged).
5. hermes-agent skill deliberate merge (new this run).

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

### 2026-09-28 09:30 — Release Watcher addendum: update driver identified (desktop
app), updates #2–#3 fired mid-watch, himalaya skill doc restored to stock

**Status:** Monitoring addendum to the 09:15 entry. No gateway/dashboard restart
performed by the watcher.

**Update driver identified (closes the 9/15 open question):** the driver is
Matt's desktop app. Timeline this morning (gui.log + hermes-update.log + ps
ppid): desktop ws connects 08:31/08:44/08:53 → `hermes update` #1 spawned
09:00:04 as a CHILD of the dashboard process (PPID 1260454), 10 s after the
08:59:54 ws accept; update #2 spawned 09:11:46 right after Matt's 09:06:34
reconnect; update #3 fired 09:21 (while this watcher run was in progress).
The "unidentified daily fast-forwards" since 9/15 are desktop-triggered
updates (Matt or the desktop app's update affordance), not cron. The three
9/28 runs logged: v0.21.5+2945 → +4108 (09:00), +4108 → +4125 (09:11),
+4125 → +4126 [main @ 96aa40a49f] (09:21). Each run also re-triggers the
multiplex recommendation + RestartPreventExitStatus=78 warning in its tail.

**Watcher script:** the parallel 09:13 fix (probe repo venv first, git-derived
behind, base-semver reporting) is correct and verified in this run — output
`latest=v0.21.5 installed=v0.21.5 behind=0`. Mirrors identical in
/root/.hermes/scripts/ and profiles/daily/scripts/.

**himalaya skill doc restored to stock (by the watcher run):**
`hermes skills list-modified` showed 2 user-modified bundled skills after the
update code drop: `hermes-agent` (38-file drift; stock is version 3.2.0 vs
our stale 2.1.0 copy — resolved by the update's bundled-skill baseline) and
`himalaya` (1 file: our copy had `himalaya message move 42 "Archive"` — ID
before target — while the installed himalaya CLI takes `<TARGET> <ID>...`,
target first). The stale local doc was factually wrong and could make Caddy
construct backwards move/copy commands; restored with
`hermes skills reset himalaya --restore` (verified: list-modified now shows
no user-modified bundled skills).

**Cron liveness false-negative:** `hermes cron status` (repo venv CLI) says
"No scheduler is serving profile 'daily' — its cron jobs will NOT fire."
Contradicted by observed reality: ticker heartbeat/.tick.lock fresh within
the last minute, and 5 job outputs written 09:00–09:15 (this watcher job
itself fired at 09:13:45 from the post-fix monitor change). The new
liveness probe appears to miss the system-unit gateway's routing
registration (same signature as the update-flow's multiplex advice).
Do not act on the ✗ line by installing a second gateway — cron is ticking.

**Gateway restart debt (unchanged, needs Matt):** gateway PID 1260418 still
running the 9/26 e295174 snapshot — three update drops (105568c, e824425e,
96aa40a) live on disk, not in gateway memory. Dashboard DID restart (09:15:48,
new PID 1464529, ExecStart intact with --open-profile daily). Post-update
CLI still prints the mixed-sys.modules warning on every invocation.

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
