# playwright-core 1.61.1 Viewport Compatibility Implementation Plan

**Date:** 2026-07-06
**Project:** camofox-browser
**Status:** Approved
<!-- Approved after second-pass self-review on 2026-07-06. -->

## Plan Review Status
- Review Status: Approved
- Reviewer Type: Second-pass self-review
- Review Date: 2026-07-06
- Blocking Issues Fixed: 0
- Review Artifact: inline approval notes below
- Approval Notes: Second-pass self-review approved the plan as execution-ready. All template sections present. Spec coverage map covers FR-001 through FR-007 and SC-001 through SC-006. Validation coverage map ties each behavior change to a concrete gate. 3 task groups with W-question matrices, stop conditions, and commit boundaries. Key insight documented: getSession creates context but newPage happens in POST /tabs handler — setViewportSize must be added there. Rollback plan concrete (5 steps). Autonomy contract defines clear stop conditions. Non-blocking: helper function vs per-callsite edits decision deferred to executor.

## Goal
Apply the `viewport: null` + `page.setViewportSize()` workaround in `server.js` and update `playwright-core` to 1.61.1 so E2E smoke tests pass without camoufox-js changes.

## Architecture Summary
The plan modifies three `browser.newContext()` call sites in `server.js` to avoid passing `viewport` at context-creation time, then calls `page.setViewportSize()` after `context.newPage()` to restore the 1280×720 default. This avoids the `Browser.setDefaultViewport` Juggler protocol command that carries the unsupported `isMobile` field in Playwright 1.61.

All three call sites overlap the same file (`server.js`), so the plan is sequential. The `playwright-core` update must happen before validation, but after the code changes to avoid a broken intermediate state.

## Technical Context
- Language/Version: JavaScript ESM on Node `>=22`
- Primary Dependencies: `camoufox-js@0.10.2`, `playwright-core@1.59.1` (target: 1.61.1), `express@4`, `swagger-jsdoc@6.2.8`
- Storage: npm package metadata only
- Testing Stack: Jest with native ESM, `jest.config.e2e.cjs` for browser smoke, `jest.config.cjs` for unit
- Platform: Linux; Camoufox headless Firefox
- Constraints: `camoufox-js` pre-1.0 with deep import; build script suppresses TS errors (`tsc -p . || true`); `package-lock.json` has local modifications in main working tree

## Input Artifacts
- Spec: `/home/jan/gh/camofox-browser/specs/2026-07-06-playwright-core-1.61.1-viewport-compat-spec.md`
- Supporting Docs: `/home/jan/gh/camofox-browser/AGENTS.md`
- Session Evidence: Predecessor commit `bbb7b44` on `deps/selective-dependency-maintenance` (rolled back 1.61.1 per FR-007); consolidated context at `local://playwright-core-1.61.1-consolidated-context.md`

## Operational State
- Current Workspace: Main working tree at `~/gh/camofox-browser` on `master` with `M package-lock.json` (local lockfile drift, pre-existing). Worktree `selective-dependency-maintenance` at `~/.omp/agent/worktrees/camofox-browser/selective-dependency-maintenance` on branch `deps/selective-dependency-maintenance` with commit `bbb7b44`.
- Current Artifact State: Approved spec at `specs/2026-07-06-playwright-core-1.61.1-viewport-compat-spec.md`; no implementation plan existed before this artifact.
- Current Session State: Predecessor dependency-maintenance slice complete (`bbb7b44`). Spec framing and approval complete. No code changes applied for this spec yet.
- Stale Review Risk: Low — spec approved 2026-07-06 in this session.

## Session Rehydration
| Session / Handover / State Source | Why Relevant | State Extracted | Confidence | Follow-up Check |
|---|---|---|---|---|
| Predecessor `deps/selective-dependency-maintenance` commit `bbb7b44` | Contains `@types/node@22.20.0` update + plugin.ts fixes already committed; this plan builds on that branch | Branch `deps/selective-dependency-maintenance` has `@types/node` slice committed; `playwright-core` still at 1.59.1 | High | Re-check branch state before execution |
| 2026-07-06 E2E smoke results | Confirmed 1.61.1 fails with `isMobile` protocol error; 1.59.1 passes 22/22 | Root cause validated | High | None |
| `daijro/camoufox#653` | Upstream confirms the incompatibility and that pinning `<1.61` is the temporary upstream fix | No Juggler patch released | High | Monitor for upstream patch |

## Execution-Review Contract
- Plan Goal: Deliver a working `playwright-core` 1.61.1 update with the viewport workaround so all E2E smoke tests pass.
- Source Contract: `specs/2026-07-06-playwright-core-1.61.1-viewport-compat-spec.md`, covering FR-001 through FR-007, SC-001 through SC-006, and all acceptance scenarios.
- Execution Scope: `/home/jan/gh/camofox-browser/server.js` (3 call sites), `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`; validation via `npx tsc -p .`, `npm run generate-openapi`, OpenAPI unit test, E2E smoke, `npm audit --json`, diff review.
- Execution Non-Scope: `camoufox-js`, Juggler binary, `swagger-jsdoc`, `express@5`, `jest@30`, `typescript@6`, unrelated refactors, new tests, plugin behavior changes.
- Dependency Order: Workspace isolation → code changes in `server.js` → `playwright-core` install → validation → commit. Sequential only.
- Validation Contract: Each gate must pass before the next. E2E smoke (22/22) is the primary gate. If E2E fails, rollback both code and dependency.
- Risk Classes: snapshot/screenshot behavior drift from `setViewportSize` vs context-level viewport; `session:creating` hook injecting `viewport` into `contextOptions`; health-probe call site (EC5) also failing; lockfile churn.

## Autonomy Contract
| Field | Required Answer |
|---|---|
| Goal | Apply viewport workaround in server.js and update playwright-core to 1.61.1 with all validation gates passing |
| Allowed Scope | `server.js`, `package.json`, `package-lock.json` in the `deps/selective-dependency-maintenance` worktree; repo-local npm and Jest commands |
| Non-Scope | `camoufox-js`, Juggler, other deps, new tests, refactors, route changes, plugin changes |
| Continuation Condition | Continue only when the current validation gate passes and the diff is limited to the allowed scope |
| Stop Condition | Stop on any validation failure, any file drift outside scope, or any sign that `camoufox-js` work is required |
| Handover Condition | Write execution-state if a slice fails and rollback is needed |
| Validation Evidence | `npx tsc -p .` exit 0; OpenAPI unit 16/16; `npm run generate-openapi` success; E2E smoke 22/22; `npm audit --json` no new vulnerabilities |
| Rollback Evidence | Pre-slice state of `server.js`, `package.json`, `package-lock.json` captured before changes |

## Execution Mode Recommendation
| Mode | Recommended? | Use When | TDD Relationship | Handoff |
|---|---|---|---|---|
| Direct TDD single-slice | No | One small implementation slice | — | `Handoff: /workflow/execution/test-driven-development [artifact=approved-plan]` |
| Controlled inline execution | Yes, after workspace isolation | Sequential task groups with heavy file overlap in `server.js` | Use TDD-style validation (E2E smoke as test-first gate) | `Handoff: /workflow/controller/executing-plans [artifact=approved-plan]` |
| Subagent-driven development | No | Not safe — all tasks overlap `server.js` | — | — |
| Parallel agent development | No | Not safe — all tasks overlap `server.js` | — | — |
| Isolated workspace first | Yes | Worktree `deps/selective-dependency-maintenance` already exists and is clean | Runs before controlled inline execution | `Handoff: /workflow/workspace/using-git-worktrees [artifact=approved-plan]` |

Recommended Mode: Use the existing `deps/selective-dependency-maintenance` worktree (already isolated), then `Controlled inline execution` because all three call sites overlap `server.js`.

## File Map
| Absolute Path | Action | Responsibility |
|---|---|---|
| /home/jan/gh/camofox-browser/server.js | Modify | Change 3 `newContext()` call sites to use `viewport: null` + `page.setViewportSize()` |
| /home/jan/gh/camofox-browser/package.json | Modify | Update `playwright-core` to `^1.61.1` |
| /home/jan/gh/camofox-browser/package-lock.json | Modify | Resolved `playwright-core` 1.61.1 |
| /home/jan/gh/camofox-browser/tsconfig.json | Validate | Type-check surface |
| /home/jan/gh/camofox-browser/plugin.ts | Validate | TypeScript compile target (already modified in predecessor commit) |
| /home/jan/gh/camofox-browser/jest.config.e2e.cjs | Validate | E2E config |
| /home/jan/gh/camofox-browser/jest.config.cjs | Validate | Unit test config |
| /home/jan/gh/camofox-browser/tests/e2e/globalSetup.js | Validate | E2E harness bootstrap |
| /home/jan/gh/camofox-browser/tests/e2e/tabLifecycle.test.js | Validate | Browser lifecycle smoke |
| /home/jan/gh/camofox-browser/tests/e2e/navigation.test.js | Validate | Navigation smoke |
| /home/jan/gh/camofox-browser/tests/e2e/snapshotScreenshot.test.js | Validate | Snapshot/screenshot smoke |
| /home/jan/gh/camofox-browser/tests/unit/openapi.test.js | Validate | OpenAPI regression gate |
| /home/jan/gh/camofox-browser/lib/openapi.js | Validate | OpenAPI generation surface |
| /home/jan/gh/camofox-browser/openapi.json | Validate | Confirm no route changes |

## Spec Coverage Map
| Spec Requirement / Scenario / Success Criterion | Covered By Task Group(s) | Notes |
|---|---|---|
| FR-001 (update playwright-core to 1.61.1) | TG2 | `npm install playwright-core@1.61.1` |
| FR-002 (modify newContext call sites) | TG1 | All 3 call sites in server.js |
| FR-003 (call setViewportSize after newPage) | TG1 | Added after `context.newPage()` at each site |
| FR-004 (preserve permissions) | TG1 | `permissions: ['geolocation']` kept in context options |
| FR-005 (preserve session:creating hook) | TG1 | Hook still receives `contextOptions`; `viewport` stripped before, `setViewportSize` after |
| FR-006 (evaluate test context at line 5968) | TG1 | Third call site — apply workaround if it fails |
| FR-007 (no camoufox-js changes) | TG1, TG2 | Only server.js and package files touched |
| SC-001 (E2E smoke 22/22) | TG3 | Primary validation gate |
| SC-002 (tsc passes) | TG3 | Type-check gate |
| SC-003 (openapi generation) | TG3 | OpenAPI generation gate |
| SC-004 (openapi unit 16/16) | TG3 | OpenAPI unit gate |
| SC-005 (audit no new vulns) | TG3 | Audit gate |
| SC-006 (server.js diff limited) | TG3 | Diff review gate |
| Acceptance 2.1.1 (E2E passes) | TG3 | SC-001 |
| Acceptance 2.1.2 (diff limited to call sites) | TG3 | SC-006 |
| Acceptance 2.1.3 (no new audit vulns) | TG3 | SC-005 |

## Validation Coverage Map
| Behavior / Risk | Validation Step | Expected Evidence |
|---|---|---|
| Viewport workaround preserves 1280×720 | E2E smoke (snapshotScreenshot.test.js) | 22/22 pass with correct viewport in snapshots |
| Type-check passes | `npx tsc -p .` | Zero exit code |
| OpenAPI generation unchanged | `npm run generate-openapi` + `tests/unit/openapi.test.js` | 16/16 pass; `openapi.json` no diff |
| playwright-core 1.61.1 doesn't break browser launch | E2E smoke (tabLifecycle.test.js) | Tab creation and lifecycle pass |
| Navigation works with workaround | E2E smoke (navigation.test.js) | Navigation, back, forward, refresh pass |
| No new vulnerabilities | `npm audit --json` | No new vulnerabilities vs committed lockfile baseline |
| Diff scope fidelity | `git diff --stat` inspection | Only server.js + package.json + package-lock.json changed |

## Task Group 1: Apply Viewport Workaround in server.js
**Goal:** Modify all three `newContext()` call sites to avoid `Browser.setDefaultViewport` by using `viewport: null` + `page.setViewportSize()`.
**Execution Mode:** Sequential — must run before TG2

**Task W-Question Matrix:**
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Executor applies code changes; reviewer checks diff | Spec FR-002, FR-003 | Yes |
| Was | Replace context-level `viewport` with `viewport: null` + post-creation `setViewportSize()` at 3 call sites | Spec FR-002, FR-003, FR-004, FR-005 | Yes |
| Wo | `/home/jan/gh/camofox-browser/server.js` lines 709, 1180-1201, 5968 | Spec E-001 | Yes |
| Wie | Edit each call site: remove `viewport` from `newContext()` options, add `page.setViewportSize()` after `newPage()` | Spec §14.1, E-002 | Yes |
| Womit | `edit` tool for surgical changes | File Map | Yes |
| Wovon | Approved spec; predecessor commit `bbb7b44` (already on branch) | Session Rehydration | Yes |
| Wogegen | Breaking `permissions` option, breaking `session:creating` hook, missing the health-probe call site | Spec EC2, EC3, EC5 | Yes |
| Warum/Wieso/Weshalb | All three call sites overlap `server.js` — must be done in one pass before installing 1.61.1 | Architecture Summary | Yes |
| Welche | Diff inspection confirms only 3 call sites changed | Spec SC-006 | Yes |

**Delegation Readiness:** single-controller only — all changes overlap `server.js`.
**Stop Conditions:** stop if any call site has dependencies that make the workaround unsafe (e.g., `viewport` used downstream for logic).

**Files:**
- Modify: `/home/jan/gh/camofox-browser/server.js`

### Call Site 1: `probeGoogleSearch` (line 709)

Current code (lines 708-713):
```js
    context = await candidateBrowser.newContext({
      viewport: { width: 1280, height: 720 },
      permissions: ['geolocation'],
    });
    const page = await context.newPage();
```

Target code:
```js
    context = await candidateBrowser.newContext({
      viewport: null,
      permissions: ['geolocation'],
    });
    const page = await context.newPage();
    await page.setViewportSize({ width: 1280, height: 720 });
```

### Call Site 2: `getSession` (lines 1179-1201)

Current code (lines 1179-1201):
```js
      const contextOptions = {
        viewport: { width: 1280, height: 720 },
        permissions: ['geolocation'],
      };
      // ... locale/timezone/geo/proxy options added conditionally ...
      await pluginEvents.emitAsync('session:creating', { userId: key, contextOptions });
      const context = await b.newContext(contextOptions);
```

Target code:
```js
      const contextOptions = {
        viewport: null,
        permissions: ['geolocation'],
      };
      // ... locale/timezone/geo/proxy options added conditionally ...
      await pluginEvents.emitAsync('session:creating', { userId: key, contextOptions });
      const context = await b.newContext(contextOptions);
```

Then after `context.newPage()` is called in the tab creation flow, `page.setViewportSize()` must be called. The tab creation happens in `app.post('/tabs', ...)` (line 2568). The first page is created there. The workaround for this call site requires finding where `context.newPage()` is first called for a new session and adding `setViewportSize` there.

**Important:** The `getSession` function creates the context but does NOT create the first page — that happens later in the `POST /tabs` handler. The `setViewportSize` call must be added in the tab creation flow, not in `getSession`. The planner must trace where `context.newPage()` is called for a new tab and add `page.setViewportSize({ width: 1280, height: 720 })` there.

Alternatively, a helper function could wrap `newContext` to always apply the workaround. This would centralize the logic and avoid scattering `setViewportSize` calls. The planner should evaluate whether a helper is cleaner than per-callsite edits.

### Call Site 3: Health probe (line 5968)

Current code (lines 5967-5970):
```js
    testContext = await browser.newContext();
    const page = await testContext.newPage();
    await page.goto('about:blank', { timeout: 5000 });
```

Target code:
```js
    testContext = await browser.newContext({ viewport: null });
    const page = await testContext.newPage();
    await page.setViewportSize({ width: 1280, height: 720 });
    await page.goto('about:blank', { timeout: 5000 });
```

Note: This call site uses no explicit `viewport` currently, but Playwright 1.61 may still send a default viewport with `isMobile`. Adding `viewport: null` explicitly is the safe path.

- [ ] Step 1: Run `git status --short` in the worktree and confirm clean state on `deps/selective-dependency-maintenance` branch.
- [ ] Step 2: Capture pre-slice state: `git show HEAD:server.js > /tmp/server.js.pre-slice` for rollback.
- [ ] Step 3: Edit Call Site 1 (`probeGoogleSearch`, line 709): replace `viewport: { width: 1280, height: 720 }` with `viewport: null` and add `await page.setViewportSize({ width: 1280, height: 720 })` after `context.newPage()`.
- [ ] Step 4: Edit Call Site 2 (`getSession`, line 1179): replace `viewport: { width: 1280, height: 720 }` with `viewport: null` in `contextOptions`.
- [ ] Step 5: Trace where `context.newPage()` is first called for a new tab in the `POST /tabs` handler. Add `await page.setViewportSize({ width: 1280, height: 720 })` after the first `newPage()` call for each new tab. If a helper function is cleaner, implement it instead and use it at all call sites.
- [ ] Step 6: Edit Call Site 3 (health probe, line 5968): add `viewport: null` to `newContext()` and add `await page.setViewportSize({ width: 1280, height: 720 })` after `newPage()`.
- [ ] Step 7: Inspect `git diff -- server.js` and verify only the 3 call sites (plus the tab-creation `setViewportSize` addition) changed.
- [ ] Step 8: Run `npx tsc -p .` and verify zero exit code.

## Task Group 2: Update playwright-core to 1.61.1
**Goal:** Install `playwright-core` 1.61.1 and verify no `camoufox-js` changes are pulled.
**Execution Mode:** Must run after TG1

**Task W-Question Matrix:**
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Executor installs; reviewer checks diff | Spec FR-001 | Yes |
| Was | Update `playwright-core` to 1.61.1 | Spec FR-001 | Yes |
| Wo | `/home/jan/gh/camofox-browser/package.json`, `package-lock.json` | File Map | Yes |
| Wie | `npm install playwright-core@1.61.1` | Spec FR-001 | Yes |
| Womit | npm | — | Yes |
| Wovon | TG1 complete (workaround in place) | Dependency Order | Yes |
| Wogegen | `camoufox-js` being pulled as transitive; lockfile churn beyond scope | Spec FR-007, NFR-001 | Yes |
| Warum/Wieso/Weshalb | Install after code changes to avoid broken intermediate state | Architecture Summary | Yes |
| Welche | `git diff -- package.json package-lock.json` shows only `playwright-core` change | Spec SC-006 | Yes |

**Delegation Readiness:** single-controller only — overlaps package files.
**Stop Conditions:** stop if `camoufox-js` appears in the diff or if unrelated direct dependencies change.

**Files:**
- Modify: `/home/jan/gh/camofox-browser/package.json`
- Modify: `/home/jan/gh/camofox-browser/package-lock.json`

- [ ] Step 1: Run `npm install playwright-core@1.61.1` from the worktree root.
- [ ] Step 2: Run `git diff -- package.json | grep -E '^[+-].*camoufox'` and verify `camoufox-js` is NOT changed.
- [ ] Step 3: Run `git diff --stat -- package.json package-lock.json` and verify the diff is scoped to `playwright-core` and its direct transitives.
- [ ] Step 4: Verify `cat node_modules/playwright-core/package.json | grep '"version"'` shows `1.61.1`.

## Task Group 3: Run All Validation Gates and Commit
**Goal:** Prove the workaround + dependency update works end-to-end and commit.
**Execution Mode:** Must run after TG2

**Task W-Question Matrix:**
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Executor runs validation; reviewer checks evidence | Spec SC-001 through SC-006 | Yes |
| Was | Run all 6 validation gates and commit if all pass | Spec §20 | Yes |
| Wo | Worktree root; all validation commands are repo-local | File Map | Yes |
| Wie | Run tsc, OpenAPI gen, OpenAPI unit, E2E smoke, audit, diff review in order | Validation Coverage Map | Yes |
| Womit | npm, npx jest, npx tsc, git | — | Yes |
| Wovon | TG1 and TG2 complete | Dependency Order | Yes |
| Wogegen | False pass, scope drift, hidden regression | Spec §21 | Yes |
| Warum/Wieso/Weshalb | All gates must pass before claiming success | Spec SC-001 through SC-006 | Yes |
| Welche | Passing command outputs + clean diff + commit hash | Validation Evidence | Yes |

**Delegation Readiness:** single-controller only — evaluates aggregate result.
**Stop Conditions:** stop if any gate fails. Rollback both TG1 and TG2 if E2E smoke fails.

**Files:**
- Validate: all files in File Map
- Commit: `server.js`, `package.json`, `package-lock.json`

- [ ] Step 1: Run `npx tsc -p .` and verify zero exit code.
- [ ] Step 2: Run `npm run generate-openapi` and verify success. Check `git diff --stat -- openapi.json` — should show no diff (no route changes).
- [ ] Step 3: Run `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.cjs --runInBand --forceExit tests/unit/openapi.test.js` and verify 16/16 pass.
- [ ] Step 4: Run `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.e2e.cjs --runInBand --forceExit tests/e2e/tabLifecycle.test.js tests/e2e/navigation.test.js tests/e2e/snapshotScreenshot.test.js` and verify 22/22 pass.
- [ ] Step 5: Run `npm audit --json` and compare vulnerability count to the committed lockfile baseline (pre-existing vulnerabilities from the committed master lockfile are out of scope; verify no NEW vulnerabilities introduced by this slice).
- [ ] Step 6: Run `git diff --stat` and verify only `server.js`, `package.json`, `package-lock.json` changed.
- [ ] Step 7: If all gates pass, commit: `git add server.js package.json package-lock.json && git commit -m "fix: workaround playwright-core 1.61.1 isMobile viewport incompatibility

Replace context-level viewport with viewport: null + page.setViewportSize()
at all three newContext() call sites to avoid Browser.setDefaultViewport
protocol error from Camoufox Juggler schema missing isMobile field.

Refs: daijro/camoufox#653
       specs/2026-07-06-playwright-core-1.61.1-viewport-compat-spec.md"`.
- [ ] Step 8: If E2E smoke (Step 4) fails, rollback: `git checkout HEAD -- server.js package.json package-lock.json && npm install` and stop. Do NOT widen scope into camoufox-js work.

## Validation Evidence
| Validation ID | Command or Check | Expected Evidence | Required Before Claim |
|---|---|---|---|
| V-001 | `npx tsc -p .` | Zero exit code | TG3 completion |
| V-002 | `npm run generate-openapi` | Success; `openapi.json` no diff | TG3 completion |
| V-003 | `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.cjs --runInBand --forceExit tests/unit/openapi.test.js` | 16/16 pass | TG3 completion |
| V-004 | `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.e2e.cjs --runInBand --forceExit tests/e2e/tabLifecycle.test.js tests/e2e/navigation.test.js tests/e2e/snapshotScreenshot.test.js` | 22/22 pass | TG3 completion |
| V-005 | `npm audit --json` | No new vulnerabilities vs baseline | TG3 completion |
| V-006 | `git diff --stat` | Only server.js + package.json + package-lock.json | TG3 completion |

## Review Evidence
| Review ID | Reviewer | Scope | Status | Blocking Issues Fixed |
|---|---|---|---|---|
| R-001 | Second-pass self-review | Full plan vs approved spec | Pending | 0 |

## Validation Strategy
- [ ] Unit: OpenAPI unit test (16/16 pass)
- [ ] Integration: OpenAPI generation (success, no diff)
- [ ] Type-check: `npx tsc -p .` (zero exit)
- [ ] E2E: Browser smoke (22/22 pass)
- [ ] Audit: `npm audit --json` (no new vulnerabilities)
- [ ] Manual: Diff review (only 3 files changed)

## Rollback Plan
- [ ] Restore `/home/jan/gh/camofox-browser/server.js` from pre-slice state: `git checkout HEAD -- server.js`
- [ ] Restore `/home/jan/gh/camofox-browser/package.json` and `package-lock.json`: `git checkout HEAD -- package.json package-lock.json`
- [ ] Run `npm install` to restore `node_modules` to 1.59.1
- [ ] Run E2E smoke to confirm 22/22 pass on 1.59.1
- [ ] Stop and do not widen scope into camoufox-js work

## Open Questions
- [ ] Should `setViewportSize` be centralized in a helper function rather than scattered per call site? Non-blocking — planner/executor can decide based on code structure clarity.
- [ ] Should `playwright-core` be pinned to `>=1.61.0 <1.62.0` instead of `^1.61.0`? Non-blocking — can be decided during execution.

## References
- `/home/jan/gh/camofox-browser/specs/2026-07-06-playwright-core-1.61.1-viewport-compat-spec.md` (approved spec)
- `/home/jan/gh/camofox-browser/specs/2026-07-04-selective-dependency-maintenance-spec.md` (predecessor spec)
- `https://github.com/daijro/camoufox/issues/653` (upstream incompatibility)
- `https://playwright.dev/docs/release-notes` (Playwright 1.61 release notes)
- `/home/jan/gh/camofox-browser/AGENTS.md` (project conventions)