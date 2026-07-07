# Selective Dependency Maintenance Implementation Plan

**Date:** 2026-07-04
**Project:** camofox-browser
**Status:** Approved
<!-- Approved after independent plan review on 2026-07-04. -->

## Plan Review Status
- Review Status: Approved
- Reviewer Type: Independent
- Review Date: 2026-07-04
- Blocking Issues Fixed: 0
- Review Artifact: `agent://PlanReview`
- Approval Notes: Independent review approved the plan as execution-ready. The plan is aligned to the approved spec, keeps all dependency slices sequential because of shared `package.json` and `package-lock.json` overlap, and uses exact validation/rollback gates for `@types/node`, `swagger-jsdoc`, and `playwright-core`.

## Goal
Execute the approved selective dependency-maintenance slice for `@types/node`, `swagger-jsdoc`, and `playwright-core` without widening into deferred migrations or losing the current workspace state.

## Architecture Summary
The plan keeps dependency maintenance bounded by treating each approved direct dependency update as its own execution slice with its own validation gate and rollback boundary. This matches the approved spec's risk model: `@types/node` is compile-surface validation, `swagger-jsdoc` is generated-artifact and OpenAPI-contract validation, and `playwright-core` is runtime smoke validation.

The main architectural constraint is file overlap. All planned slices touch `/home/jan/gh/camofox-browser/package.json` and `/home/jan/gh/camofox-browser/package-lock.json`, and the `swagger-jsdoc` slice also regenerates `/home/jan/gh/camofox-browser/openapi.json`. Because of that overlap, this plan is intentionally sequential and not parallel-safe. The current workspace is also dirty, so the first safe move before any execution is isolation or an explicit state-capture boundary.

## Technical Context
- Language/Version: JavaScript ESM on Node `>=22`; limited TypeScript validation surface through `/home/jan/gh/camofox-browser/plugin.ts`
- Primary Dependencies: `camoufox-js`, `express`, `playwright-core`, `swagger-jsdoc`, `prom-client`
- Storage: npm dependency metadata plus generated `/home/jan/gh/camofox-browser/openapi.json`
- Testing Stack: Jest with native ESM, `/home/jan/gh/camofox-browser/jest.config.cjs`, and `/home/jan/gh/camofox-browser/jest.config.e2e.cjs`
- Platform: Linux workspace at `/home/jan/gh/camofox-browser`
- Constraints: current workspace is dirty; `/home/jan/gh/camofox-browser/package-lock.json` is already modified; `camoufox-js` and all major migrations remain out of scope; browser-runtime validation depends on e2e global setup/teardown

## Input Artifacts
- Spec: `/home/jan/gh/camofox-browser/specs/2026-07-04-selective-dependency-maintenance-spec.md`
- Supporting Docs: `/home/jan/gh/camofox-browser/AGENTS.md`
- Session Evidence: `N/A — current repo state and approved spec are sufficient for planning`

## Operational State
- Current Workspace: `/home/jan/gh/camofox-browser` with `M package-lock.json` and untracked `specs/2026-07-04-selective-dependency-maintenance-spec.md`
- Current Artifact State: approved spec exists at `/home/jan/gh/camofox-browser/specs/2026-07-04-selective-dependency-maintenance-spec.md`; no implementation plan existed before this artifact
- Current Session State: dependency audit, scoped spec drafting, and independent spec review are complete; no dependency updates have been executed in this session
- Stale Review Risk: low for the spec because it was independently approved on 2026-07-04; medium for execution if workspace state changes before implementation begins

## Session Rehydration
| Session / Handover / State Source | Why Relevant | State Extracted | Confidence | Follow-up Check |
|---|---|---|---|---|
| Current 2026-07-04 dependency-maintenance session | The plan continues directly from the approved spec created from current findings | Clean audit posture, scoped dependency target set, deferred migrations, dirty workspace warning | High | Re-check workspace state immediately before execution |
| `git status --short -- package-lock.json package.json openapi.json specs/2026-07-04-selective-dependency-maintenance-spec.md` | Execution safety depends on not overwriting current local state | `package-lock.json` already modified; spec file untracked | High | Re-run status after any workspace-isolation step |

## Execution-Review Contract
- Plan Goal: deliver the approved selective dependency-maintenance slice with exact validation and rollback boundaries
- Source Contract: `/home/jan/gh/camofox-browser/specs/2026-07-04-selective-dependency-maintenance-spec.md`, covering FR-001 through FR-007, SC-001 through SC-006, and both acceptance-scenario groups
- Execution Scope: `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, `/home/jan/gh/camofox-browser/openapi.json`, `/home/jan/gh/camofox-browser/lib/openapi.js`, `/home/jan/gh/camofox-browser/scripts/generate-openapi.js`, `/home/jan/gh/camofox-browser/tests/unit/openapi.test.js`, `/home/jan/gh/camofox-browser/tsconfig.json`, `/home/jan/gh/camofox-browser/plugin.ts`, `/home/jan/gh/camofox-browser/server.js`, `/home/jan/gh/camofox-browser/jest.config.cjs`, `/home/jan/gh/camofox-browser/jest.config.e2e.cjs`, `/home/jan/gh/camofox-browser/tests/e2e/globalSetup.js`, `/home/jan/gh/camofox-browser/tests/e2e/tabLifecycle.test.js`, `/home/jan/gh/camofox-browser/tests/e2e/navigation.test.js`, `/home/jan/gh/camofox-browser/tests/e2e/snapshotScreenshot.test.js`
- Execution Non-Scope: `/home/jan/gh/camofox-browser/node_modules/` manual edits, `camoufox-js`, `express@5`, `jest@30`, `typescript@6`, unrelated refactors, runtime feature changes, and any new test expansion beyond the approved smoke gates
- Dependency Order: workspace isolation/state capture first; then `@types/node`; then `swagger-jsdoc`; then `playwright-core`; then final scope/diff review
- Validation Contract: each slice must pass its own command gate before the next slice begins; unexpected file drift or validation failures trigger stop-and-rollback for the current slice
- Risk Classes: lockfile overwrite, runtime browser regression, generated OpenAPI drift, compile drift hidden by the build script, accidental scope widening into deferred migrations

## Autonomy Contract
| Field | Required Answer |
|---|---|
| Goal | If execution is later authorized, apply the approved dependency slices only while preserving the current workspace state and deferral boundaries |
| Allowed Scope | Only the files listed in the Execution Scope; repo-local npm commands; repo-local Jest/TypeScript/OpenAPI validation commands |
| Non-Scope | `camoufox-js`, `express@5`, `jest@30`, `typescript@6`, unrelated cleanup, broad `npm update`, `npm audit fix`, or ad hoc refactors |
| Continuation Condition | Continue automatically only when the current slice passes validation and the diff remains inside the allowed file map |
| Stop Condition | Stop immediately on failed validation, unexpected file drift, any sign that `camoufox-js` work is required, or any change outside the allowed scope |
| Handover Condition | Write execution-state or return to planning if a slice fails or if workspace isolation cannot preserve the current local state |
| Validation Evidence | Exact commands in Task Groups 2-5 plus final diff review |
| Rollback Evidence | Pre-slice state capture for `package.json`, `package-lock.json`, and `openapi.json`, plus per-slice rollback steps |

## Execution Mode Recommendation
| Mode | Recommended? | Use When | TDD Relationship | Handoff |
|---|---|---|---|---|
| Direct TDD single-slice | No | One small implementation slice | This mode is the TDD workflow | `Handoff: /workflow/execution/test-driven-development [artifact=approved-plan]` |
| Controlled inline execution | Yes, after workspace isolation | Sequential task groups with heavy file overlap | Use TDD-style validation inside each behavior-changing slice | `Handoff: /workflow/controller/executing-plans [artifact=approved-plan]` |
| Subagent-driven development | No | Mostly independent task groups needing per-task reintegration and review | Each worker uses TDD for behavior-changing slices | `Handoff: /workflow/controller/subagent-driven-development [artifact=approved-plan]` |
| Parallel agent development | No | Truly independent domains with no unstable file overlap | Each worker uses TDD for behavior-changing slices | `Handoff: /workflow/controller/dispatching-parallel-agents [artifact=approved-plan]` |
| Isolated workspace first | Yes | Current workspace is dirty, shared, or unsafe | Runs before the selected execution mode | `Handoff: /workflow/workspace/using-git-worktrees [artifact=approved-plan]` |

Recommended Mode: `Isolated workspace first`, followed by `Controlled inline execution`, because the current workspace is dirty and every implementation slice overlaps the same package and lockfile state.

## File Map
| Absolute Path | Action | Responsibility |
|---|---|---|
| /home/jan/gh/camofox-browser/package.json | Modify | Direct dependency range updates if npm rewrites current ranges |
| /home/jan/gh/camofox-browser/package-lock.json | Modify | Resolved dependency updates for every approved slice |
| /home/jan/gh/camofox-browser/openapi.json | Generate | Regenerated artifact after the `swagger-jsdoc` slice |
| /home/jan/gh/camofox-browser/lib/openapi.js | Validate; modify only if `swagger-jsdoc` compatibility drift requires it | OpenAPI generation surface |
| /home/jan/gh/camofox-browser/scripts/generate-openapi.js | Validate; modify only if `swagger-jsdoc` compatibility drift requires it | OpenAPI generation entrypoint |
| /home/jan/gh/camofox-browser/tests/unit/openapi.test.js | Validate; modify only if compatibility assertions need a narrow update | Swagger/OpenAPI regression gate |
| /home/jan/gh/camofox-browser/tsconfig.json | Validate | Type-check surface definition |
| /home/jan/gh/camofox-browser/plugin.ts | Validate | Current TypeScript compile target |
| /home/jan/gh/camofox-browser/server.js | Validate; modify only if `playwright-core` compatibility drift requires it | Browser runtime surface |
| /home/jan/gh/camofox-browser/jest.config.cjs | Validate | Unit-test config used by the OpenAPI validation gate |
| /home/jan/gh/camofox-browser/jest.config.e2e.cjs | Validate | E2E config used by the `playwright-core` smoke gate |
| /home/jan/gh/camofox-browser/tests/e2e/globalSetup.js | Validate | Required e2e harness bootstrap for the `playwright-core` gate |
| /home/jan/gh/camofox-browser/tests/e2e/tabLifecycle.test.js | Validate | Browser lifecycle smoke evidence |
| /home/jan/gh/camofox-browser/tests/e2e/navigation.test.js | Validate | Browser navigation smoke evidence |
| /home/jan/gh/camofox-browser/tests/e2e/snapshotScreenshot.test.js | Validate | Browser snapshot/screenshot smoke evidence |

## Spec Coverage Map
| Spec Requirement / Scenario / Success Criterion | Covered By Task Group(s) | Notes |
|---|---|---|
| Acceptance Scenario 2.1.1 | Task Groups 2, 3, 4, 5 | Only approved direct dependencies are updated, with drift review at the end |
| Acceptance Scenario 2.1.2 | Task Groups 2, 3, 4 | Each slice has a concrete validation gate |
| Acceptance Scenario 2.2.1 | Task Group 5 | Deferred migrations remain deferred and are re-asserted in final review |
| Acceptance Scenario 2.2.2 | Task Groups 2, 3, 4 | Every slice has an explicit stop-and-rollback path |
| FR-001 | Task Group 2 | Preflight explicitly forbids `npm audit fix` |
| FR-002 | Task Groups 2, 3, 4 | One task group per approved direct dependency slice |
| FR-003 | Task Groups 2, 3, 4 | Sequential isolated slices |
| FR-004 | Task Group 2 | Node 22 patch update and type-check validation |
| FR-005 | Task Group 3 | OpenAPI regeneration gate |
| FR-006 | Task Group 5 | Final diff review records deferral boundaries |
| FR-007 | Task Group 4 | `playwright-core` slice stops if `camoufox-js` work appears necessary |
| SC-001 | Task Group 5 | Final audit confirmation |
| SC-002 | Task Group 2 | Direct type-check evidence |
| SC-003 | Task Group 3 | OpenAPI regeneration evidence |
| SC-004 | Task Group 3 | OpenAPI unit-test evidence |
| SC-005 | Task Group 4 | E2E smoke evidence |
| SC-006 | Task Group 5 | Final scope/diff review |

## Validation Coverage Map
| Behavior / Risk | Validation Step | Expected Evidence |
|---|---|---|
| `@types/node` update does not break current TS surface | `npx tsc -p .` | Zero-exit type-check covering `/home/jan/gh/camofox-browser/plugin.ts` |
| `swagger-jsdoc` update still generates valid spec output | `npm run generate-openapi` | Successful run and regenerated `/home/jan/gh/camofox-browser/openapi.json` |
| `swagger-jsdoc` update does not break route/spec coverage | `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.cjs --runInBand --forceExit tests/unit/openapi.test.js` | Passing OpenAPI unit test |
| `playwright-core` update preserves core browser flows | `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.e2e.cjs --runInBand --forceExit tests/e2e/tabLifecycle.test.js tests/e2e/navigation.test.js tests/e2e/snapshotScreenshot.test.js` | Passing e2e smoke tests with global setup/teardown |
| Scope did not widen beyond approved slice | Inspect diff for `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, `/home/jan/gh/camofox-browser/openapi.json` | Only approved direct packages plus directly caused transitive churn |
| Audit posture remains clean | `npm audit --json` | Zero vulnerabilities |

## Task Group 1: Workspace Isolation and Baseline Capture
**Goal:** Preserve current local state before any dependency execution begins.
**Execution Mode:** Sequential

**Task W-Question Matrix:**
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Executor is the implementation agent; reviewer confirms state-capture evidence | Approved spec, Operational State | Yes |
| Was | Establish a safe execution boundary without modifying dependency state yet | Operational State | Yes |
| Wo | `/home/jan/gh/camofox-browser`, especially `package.json`, `package-lock.json`, `openapi.json` | File Map | Yes |
| Wie | Capture current state and choose isolated workspace if current workspace remains dirty | Operational State, Validation Coverage Map | Yes |
| Womit | git/worktree tooling, repo-local file reads, diff inspection | Execution-Review Contract | Yes |
| Wovon | Current dirty workspace and approved spec | Session Rehydration | Yes |
| Wogegen | Accidental overwrite of user changes, stale review assumptions, unsafe execution in a dirty tree | Operational State | Yes |
| Warum/Wieso/Weshalb | All later slices overlap the same package files, so isolation precedes mutation | Architecture Summary | Yes |
| Welche | Fresh workspace-state evidence before any package install | Validation Coverage Map | Yes |

**Delegation Readiness:** single-controller only; do not delegate because this group establishes the shared execution baseline.
**Stop Conditions:** stop if current workspace state cannot be preserved or isolated safely.

**Files:**
- Modify: `/home/jan/gh/camofox-browser/package.json` (state capture only in this group)
- Modify: `/home/jan/gh/camofox-browser/package-lock.json` (state capture only in this group)
- Generate: `/home/jan/gh/camofox-browser/openapi.json` (state capture only in this group)

- [ ] Step 1: Run `git status --short -- package-lock.json package.json openapi.json specs/2026-07-04-selective-dependency-maintenance-spec.md plans/2026-07-04-selective-dependency-maintenance-implementation-plan.md` and record the exact pre-execution state.
- [ ] Step 2: If the workspace remains dirty, hand off to `Handoff: /workflow/workspace/using-git-worktrees [artifact=approved-plan]` before any dependency install.
- [ ] Step 3: Capture the pre-slice contents or diff state for `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, and `/home/jan/gh/camofox-browser/openapi.json` so rollback is exact.
- [ ] Step 4: Confirm that `camoufox-js`, `express@5`, `jest@30`, and `typescript@6` remain out of scope before proceeding.
- [ ] Step 5: Commit `chore(plan): capture dependency maintenance baseline` only if the selected execution workflow requires an explicit baseline commit in the isolated workspace.

## Task Group 2: Update `@types/node` and Validate the Type Surface
**Goal:** Refresh the Node 22 type patch without widening into compiler or runtime migrations.
**Execution Mode:** Must run after Task Group 1

**Task W-Question Matrix:**
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Executor applies the slice; reviewer checks type evidence | Approved spec FR-002, SC-002 | Yes |
| Was | Update only `@types/node` to the latest Node 22 patch and validate the current compile target | Spec Coverage Map | Yes |
| Wo | `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, `/home/jan/gh/camofox-browser/tsconfig.json`, `/home/jan/gh/camofox-browser/plugin.ts` | File Map | Yes |
| Wie | Run a targeted npm install, then run `npx tsc -p .` | Validation Coverage Map | Yes |
| Womit | npm and TypeScript compiler | Execution-Review Contract | Yes |
| Wovon | Clean baseline from Task Group 1 | Dependency Order | Yes |
| Wogegen | Hidden type drift and accidental major type/runtime baseline jumps | Technical Context | Yes |
| Warum/Wieso/Weshalb | This is the smallest-risk slice and should fail fast before runtime updates | Architecture Summary | Yes |
| Welche | Successful targeted install plus passing type-check | Validation Coverage Map | Yes |

**Delegation Readiness:** not safe for delegation; overlaps `package.json` and `package-lock.json` with all later slices.
**Stop Conditions:** stop if npm proposes unrelated direct dependency changes or if `npx tsc -p .` fails.

**Files:**
- Modify: `/home/jan/gh/camofox-browser/package.json`
- Modify: `/home/jan/gh/camofox-browser/package-lock.json`
- Test: `/home/jan/gh/camofox-browser/tsconfig.json`
- Test: `/home/jan/gh/camofox-browser/plugin.ts`

- [ ] Step 1: Run `npm install --save-dev @types/node@22.20.0` from `/home/jan/gh/camofox-browser`.
- [ ] Step 2: Inspect the resulting diff and verify that direct dependency changes are limited to `/home/jan/gh/camofox-browser/package.json` and `/home/jan/gh/camofox-browser/package-lock.json` for `@types/node`.
- [ ] Step 3: Run `npx tsc -p .` and verify a zero exit code.
- [ ] Step 4: If Step 3 fails, restore the pre-slice state captured in Task Group 1 and stop.
- [ ] Step 5: Commit `chore(deps): update @types/node to 22.20.0`.

## Task Group 3: Update `swagger-jsdoc`, Regenerate OpenAPI, and Validate Coverage
**Goal:** Refresh `swagger-jsdoc` while proving that generated OpenAPI output and route coverage stay intact.
**Execution Mode:** Must run after Task Group 2

**Task W-Question Matrix:**
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Executor applies the slice; reviewer validates generated artifact and unit evidence | Approved spec FR-002, FR-005, SC-003, SC-004 | Yes |
| Was | Update `swagger-jsdoc`, regenerate `/home/jan/gh/camofox-browser/openapi.json`, and validate route/spec coverage | Spec Coverage Map | Yes |
| Wo | `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, `/home/jan/gh/camofox-browser/openapi.json`, `/home/jan/gh/camofox-browser/lib/openapi.js`, `/home/jan/gh/camofox-browser/scripts/generate-openapi.js`, `/home/jan/gh/camofox-browser/tests/unit/openapi.test.js` | File Map | Yes |
| Wie | Targeted npm install, OpenAPI regeneration, then focused Jest unit validation | Validation Coverage Map | Yes |
| Womit | npm, `npm run generate-openapi`, repo-local Jest config | Execution-Review Contract | Yes |
| Wovon | Successful completion of Task Group 2 | Dependency Order | Yes |
| Wogegen | Transitive parser/glob drift silently breaking generated API docs | Risks and Mitigations | Yes |
| Warum/Wieso/Weshalb | This slice changes generated output and should stabilize before runtime browser testing | Architecture Summary | Yes |
| Welche | Regenerated OpenAPI file plus passing OpenAPI unit test | Validation Coverage Map | Yes |

**Delegation Readiness:** not safe for delegation; overlaps `package.json`, `package-lock.json`, and generated artifact state.
**Stop Conditions:** stop if `npm run generate-openapi` fails, if the focused unit test fails, or if code changes spill beyond the OpenAPI surface without a narrow compatibility explanation.

**Files:**
- Modify: `/home/jan/gh/camofox-browser/package.json`
- Modify: `/home/jan/gh/camofox-browser/package-lock.json`
- Generate: `/home/jan/gh/camofox-browser/openapi.json`
- Test: `/home/jan/gh/camofox-browser/lib/openapi.js`
- Test: `/home/jan/gh/camofox-browser/scripts/generate-openapi.js`
- Test: `/home/jan/gh/camofox-browser/tests/unit/openapi.test.js`

- [ ] Step 1: Run `npm install swagger-jsdoc@6.3.0` from `/home/jan/gh/camofox-browser`.
- [ ] Step 2: Inspect the resulting diff and verify that direct dependency changes are limited to the `swagger-jsdoc` slice plus its generated/transitive consequences.
- [ ] Step 3: Run `npm run generate-openapi` and verify that `/home/jan/gh/camofox-browser/openapi.json` is regenerated successfully.
- [ ] Step 4: Run `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.cjs --runInBand --forceExit tests/unit/openapi.test.js` and verify a passing result.
- [ ] Step 5: If Step 3 or Step 4 fails, restore the pre-slice state for `package.json`, `package-lock.json`, and `openapi.json`, then stop.
- [ ] Step 6: Commit `chore(deps): update swagger-jsdoc to 6.3.0`.

## Task Group 4: Update `playwright-core` and Run E2E Browser Smoke Validation
**Goal:** Refresh `playwright-core` without widening into `camoufox-js` work.
**Execution Mode:** Must run after Task Group 3

**Task W-Question Matrix:**
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Executor applies the slice; reviewer validates runtime smoke evidence | Approved spec FR-002, FR-007, SC-005 | Yes |
| Was | Update `playwright-core` and prove that tab lifecycle, navigation, and snapshot/screenshot flows still work | Spec Coverage Map | Yes |
| Wo | `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, `/home/jan/gh/camofox-browser/server.js`, `/home/jan/gh/camofox-browser/jest.config.e2e.cjs`, `/home/jan/gh/camofox-browser/tests/e2e/globalSetup.js`, `/home/jan/gh/camofox-browser/tests/e2e/tabLifecycle.test.js`, `/home/jan/gh/camofox-browser/tests/e2e/navigation.test.js`, `/home/jan/gh/camofox-browser/tests/e2e/snapshotScreenshot.test.js` | File Map | Yes |
| Wie | Targeted npm install, then focused e2e smoke validation using the dedicated e2e Jest config | Validation Coverage Map | Yes |
| Womit | npm, repo-local e2e Jest config, e2e global setup/teardown harness | Execution-Review Contract | Yes |
| Wovon | Successful completion of Task Group 3 and preserved deferred boundary around `camoufox-js` | Dependency Order | Yes |
| Wogegen | Hidden browser-runtime regressions and accidental `camoufox-js` scope expansion | Risks and Mitigations | Yes |
| Warum/Wieso/Weshalb | Runtime browser validation is the highest-risk approved slice and belongs after lower-risk slices are stable | Architecture Summary | Yes |
| Welche | Passing focused e2e smoke suite or explicit rollback evidence | Validation Coverage Map | Yes |

**Delegation Readiness:** not safe for delegation; overlaps shared package files and depends on the exact e2e harness.
**Stop Conditions:** stop if npm proposes `camoufox-js` changes, if the e2e smoke suite fails, or if compatibility work extends beyond narrow `playwright-core` handling.

**Files:**
- Modify: `/home/jan/gh/camofox-browser/package.json`
- Modify: `/home/jan/gh/camofox-browser/package-lock.json`
- Test: `/home/jan/gh/camofox-browser/server.js`
- Test: `/home/jan/gh/camofox-browser/jest.config.e2e.cjs`
- Test: `/home/jan/gh/camofox-browser/tests/e2e/globalSetup.js`
- Test: `/home/jan/gh/camofox-browser/tests/e2e/tabLifecycle.test.js`
- Test: `/home/jan/gh/camofox-browser/tests/e2e/navigation.test.js`
- Test: `/home/jan/gh/camofox-browser/tests/e2e/snapshotScreenshot.test.js`

- [ ] Step 1: Run `npm install playwright-core@1.61.1` from `/home/jan/gh/camofox-browser`.
- [ ] Step 2: Inspect the resulting diff and verify that the direct dependency change is limited to `playwright-core` and that no explicit `camoufox-js` update was introduced.
- [ ] Step 3: Run `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.e2e.cjs --runInBand --forceExit tests/e2e/tabLifecycle.test.js tests/e2e/navigation.test.js tests/e2e/snapshotScreenshot.test.js` and verify a passing result.
- [ ] Step 4: If Step 3 fails because the failure implies `camoufox-js` coupling or broader runtime drift, restore the pre-slice state for `package.json` and `package-lock.json`, then stop instead of widening scope.
- [ ] Step 5: Commit `chore(deps): update playwright-core to 1.61.1`.

## Task Group 5: Final Scope Review, Audit Confirmation, and Deferral Recording
**Goal:** Confirm that the finished dependency slice stayed inside the approved scope and that deferred migrations remain explicitly deferred.
**Execution Mode:** Must run after Task Group 4

**Task W-Question Matrix:**
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Executor performs the final verification; reviewer checks final evidence package | Approved spec FR-006, SC-001, SC-006 | Yes |
| Was | Reconfirm audit posture, diff scope, and deferral boundaries | Spec Coverage Map | Yes |
| Wo | `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, `/home/jan/gh/camofox-browser/openapi.json` and the execution review artifact/PR description | File Map | Yes |
| Wie | Run final audit, inspect final diff, and record explicit deferral notes | Validation Coverage Map | Yes |
| Womit | npm audit output, diff inspection, execution notes/PR description | Execution-Review Contract | Yes |
| Wovon | All prior slices complete or appropriately rolled back | Dependency Order | Yes |
| Wogegen | Silent scope drift, false completion claims, and accidental migration widening | Risks and Mitigations | Yes |
| Warum/Wieso/Weshalb | Final scope and audit review is required before any claim of success | Validation Contract | Yes |
| Welche | Clean audit output, bounded diff, and explicit deferral record | Validation Coverage Map | Yes |

**Delegation Readiness:** single-controller only; this group evaluates the aggregate result.
**Stop Conditions:** stop if final audit is non-clean or if the final diff includes out-of-scope direct dependency changes.

**Files:**
- Modify: `/home/jan/gh/camofox-browser/package.json`
- Modify: `/home/jan/gh/camofox-browser/package-lock.json`
- Modify: `/home/jan/gh/camofox-browser/openapi.json`

- [ ] Step 1: Run `npm audit --json` and verify zero vulnerabilities.
- [ ] Step 2: Inspect the final diff for `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, and `/home/jan/gh/camofox-browser/openapi.json` and verify that only the approved direct dependencies and their direct consequences changed.
- [ ] Step 3: Record explicit deferral notes for `camoufox-js`, `express@5`, `jest@30`, and `typescript@6` in the execution notes or PR description.
- [ ] Step 4: If Step 1 or Step 2 fails, stop and restore the failing slice rather than claiming completion.
- [ ] Step 5: Commit `chore(deps): finalize selective dependency maintenance slice` only after the final review passes.

## Validation Evidence
| Validation ID | Command or Check | Expected Evidence | Required Before Claim |
|---|---|---|---|
| V-001 | `git status --short -- package-lock.json package.json openapi.json specs/2026-07-04-selective-dependency-maintenance-spec.md plans/2026-07-04-selective-dependency-maintenance-implementation-plan.md` | Exact pre-execution workspace state captured | Task Group 1 completion |
| V-002 | `npx tsc -p .` | Zero exit code | Task Group 2 completion |
| V-003 | `npm run generate-openapi` | Successful run and regenerated `/home/jan/gh/camofox-browser/openapi.json` | Task Group 3 completion |
| V-004 | `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.cjs --runInBand --forceExit tests/unit/openapi.test.js` | Passing OpenAPI unit test | Task Group 3 completion |
| V-005 | `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.e2e.cjs --runInBand --forceExit tests/e2e/tabLifecycle.test.js tests/e2e/navigation.test.js tests/e2e/snapshotScreenshot.test.js` | Passing e2e smoke suite | Task Group 4 completion |
| V-006 | `npm audit --json` | Zero vulnerabilities | Final completion claim |
| V-007 | Final diff review of `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, `/home/jan/gh/camofox-browser/openapi.json` | Scope-limited diff and preserved deferrals | Final completion claim |

## Review Evidence
| Review ID | Reviewer | Scope | Status | Blocking Issues Fixed |
|---|---|---|---|---|
| R-001 | Independent reviewer | Saved plan against approved spec and repo evidence | Approved | 0 |

## Validation Strategy
- [ ] Unit: pass `/home/jan/gh/camofox-browser/tests/unit/openapi.test.js` under `/home/jan/gh/camofox-browser/jest.config.cjs`
- [ ] Integration: pass `npm run generate-openapi`
- [ ] Type-check: pass `npx tsc -p .`
- [ ] E2E: pass the three targeted e2e smoke tests under `/home/jan/gh/camofox-browser/jest.config.e2e.cjs`
- [ ] Manual: review final dependency diff and confirm deferred migrations remain deferred

## Rollback Plan
- [ ] Restore the pre-slice state captured in Task Group 1 before undoing a failed slice.
- [ ] Revert `/home/jan/gh/camofox-browser/package.json` and `/home/jan/gh/camofox-browser/package-lock.json` for any failed dependency slice.
- [ ] Revert `/home/jan/gh/camofox-browser/openapi.json` when undoing a failed `swagger-jsdoc` slice.
- [ ] Stop after the failed slice and reopen planning/spec work if the failure implies `camoufox-js` or major-migration scope.

## Open Questions
None

## References
- `/home/jan/gh/camofox-browser/specs/2026-07-04-selective-dependency-maintenance-spec.md`
- `/home/jan/gh/camofox-browser/AGENTS.md`
- `/home/jan/gh/camofox-browser/package.json`
- `/home/jan/gh/camofox-browser/tsconfig.json`
- `/home/jan/gh/camofox-browser/jest.config.cjs`
- `/home/jan/gh/camofox-browser/jest.config.e2e.cjs`
- `/home/jan/gh/camofox-browser/server.js`
- `/home/jan/gh/camofox-browser/lib/openapi.js`
- `/home/jan/gh/camofox-browser/scripts/generate-openapi.js`
- `/home/jan/gh/camofox-browser/tests/unit/openapi.test.js`
- `/home/jan/gh/camofox-browser/tests/e2e/globalSetup.js`
- `/home/jan/gh/camofox-browser/tests/e2e/sharedEnv.js`
- `/home/jan/gh/camofox-browser/tests/e2e/tabLifecycle.test.js`
- `/home/jan/gh/camofox-browser/tests/e2e/navigation.test.js`
- `/home/jan/gh/camofox-browser/tests/e2e/snapshotScreenshot.test.js`
