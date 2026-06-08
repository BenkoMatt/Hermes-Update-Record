# Changelog

Dated record of Hermes Agent upgrades applied to a single production
host. See [`README.md`](./README.md) for format and privacy policy.

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
