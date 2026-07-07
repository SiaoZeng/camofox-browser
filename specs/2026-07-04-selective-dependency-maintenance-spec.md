# Selective Dependency Maintenance — Full Spec

**Date:** 2026-07-04
**Project:** camofox-browser
**Status:** Approved
<!-- Approved after independent review on 2026-07-04. -->

## Spec Review Status
- Review Status: Approved
- Reviewer Type: Independent
- Review Date: 2026-07-04
- Blocking Issues Fixed: 2
- Review Artifact: `agent://SpecReview3`
- Approval Notes: Independent review approved the spec as planning-ready after two blocking Jest-command issues were corrected. The final saved commands use repo-local Jest resolution, the dedicated e2e config for browser-smoke validation, and `env CI=` to avoid the undeclared optional `jest-junit` reporter path during command execution.

## 1. Context and Scope
The repository is a Node 22+ Express-based browser automation server with direct runtime coupling to `camoufox-js`, `playwright-core`, and OpenAPI generation via `swagger-jsdoc`.

Current inspection established that the repository is not in a security-remediation state: `npm audit --json` reported zero vulnerabilities, and `npm audit fix --dry-run --json` reported zero changes. The maintenance need is therefore dependency freshness, not emergency patching.

Broad package refresh is explicitly out of scope for this spec. `npm update --dry-run --json` would add 11 packages, change 34, and remove 10, including runtime-affecting transitive changes under `camoufox-js` and `swagger-jsdoc`. This spec defines a selective, planning-safe maintenance slice instead.

### Review Contract
- **Review goal:** approve a planning-ready, low-churn dependency maintenance spec that updates safe or bounded direct dependencies and explicitly defers risky migrations.
- **Review scope:** direct dependency and devDependency maintenance for `@types/node`, `swagger-jsdoc`, and `playwright-core`; validation strategy; rollback; explicit deferrals for `camoufox-js`, `express@5`, `jest@30`, and `typescript@6`.
- **Review non-scope:** feature work, refactors unrelated to dependency compatibility, broad transitive refresh, framework-major migrations, or opportunistic runtime hardening.
- **Success criteria:** a planner can derive a safe implementation order, exact validation steps, rollback steps, and deferral boundaries without reading chat history.
- **Risk classes:** runtime browser regressions, OpenAPI generation drift, type-check drift, lockfile churn, accidental widening into major migrations.
- **Primary evidence sources:** local repo files, current dependency commands, and official package migration/release documentation listed in References.

## 2. User Scenarios and Testing
### 2.1 User Story 1 — Refresh safe direct dependencies without widening scope (Priority: P1)
A maintainer wants the repository to stop carrying stale direct dependencies where the update is bounded and defensible.

**Why this priority:** This yields the most maintenance value with the lowest migration risk.

**Independent Test:** targeted dependency update plus targeted validation commands.

**Acceptance Scenarios:**
1. **Given** the current repository state with a clean audit and outdated direct packages, **When** the maintainer applies the scoped maintenance slice, **Then** only the spec-approved packages are updated and the repository does not silently absorb broad transitive churn unrelated to that slice.
2. **Given** the scoped update plan, **When** the maintainer runs post-update validation, **Then** type-check, OpenAPI generation, and selected browser-smoke tests pass or the maintainer rolls the slice back.

### 2.2 User Story 2 — Defer risky migrations explicitly instead of mixing them into maintenance (Priority: P1)
A maintainer wants the repository to avoid accidental major-version migrations or pre-1.0 runtime coupling changes while performing routine dependency maintenance.

**Why this priority:** The largest technical risk in the findings comes from coupling and migration breadth, not from missing security fixes.

**Independent Test:** review of resulting `package.json`/`package-lock.json` diff plus unchanged deferral list.

**Acceptance Scenarios:**
1. **Given** the current findings, **When** the maintainer completes the selective update slice, **Then** `camoufox-js`, `express@5`, `jest@30`, and `typescript@6` remain deferred unless a separate approved migration spec exists.
2. **Given** a validation failure during the selective slice, **When** the failure implies `camoufox-js` compatibility work or framework-major migration work, **Then** the maintainer stops, restores the prior dependency state, and opens a dedicated follow-up spec rather than widening this slice.

## 3. Edge Cases
- EC1: `/home/jan/gh/camofox-browser/package-lock.json` is already locally modified; implementation must avoid overwriting unrelated user work.
- EC2: `camoufox-js` is pre-1.0 and the repository uses a deep import from `camoufox-js/dist/virtdisplay.js`; runtime compatibility cannot be treated as semver-stable by assumption.
- EC3: `swagger-jsdoc` minor refresh replaces parser/glob transitive packages and may change OpenAPI generation behavior without source edits.
- EC4: `playwright-core` is a runtime dependency, so even a minor bump requires browser-smoke validation rather than type-only confidence.
- EC5: The project build script suppresses TypeScript failure via `tsc -p . || true`; validation must use `npx tsc -p .` directly.

## 4. Goals
- G1: Update bounded direct dependencies that have clear value and controlled verification cost.
- G2: Preserve runtime browser behavior and OpenAPI generation behavior during maintenance.
- G3: Keep dependency maintenance intentionally narrower than framework-major or runtime-coupling migrations.
- G4: Produce an implementation-ready artifact that can be handed to planning without hidden assumptions.

## 5. Non-Goals
- NG1: Do not use `npm audit fix` or `npm audit fix --force` as a maintenance strategy.
- NG2: Do not use a broad `npm update` across the repository.
- NG3: Do not upgrade `express` from v4 to v5 in this maintenance slice.
- NG4: Do not upgrade `jest` from v29 to v30 in this maintenance slice.
- NG5: Do not upgrade `typescript` from v5 to v6 or `@types/node` from Node 22 types to Node 26 types in this maintenance slice.
- NG6: Do not upgrade `camoufox-js` from `0.10.2` to `0.11.1` in this maintenance slice.
- NG7: Do not bundle unrelated refactors, test rewrites, or plugin behavior changes into this work.

## 6. Requirements
### 6.1 Functional Requirements
- FR-001: The implementation MUST avoid `npm audit fix` because the current audit posture is already clean and the command produces no direct changes.
- FR-002: The implementation MUST update only the approved direct dependencies in this slice: `@types/node` to the latest Node 22 patch, `swagger-jsdoc` to `6.3.0`, and `playwright-core` to `1.61.1`.
- FR-003: The implementation MUST apply dependency updates in isolated steps so that failures can be attributed to one slice at a time.
- FR-004: The implementation MUST preserve the existing declared Node engine floor of `>=22`.
- FR-005: The implementation MUST regenerate `/home/jan/gh/camofox-browser/openapi.json` after the `swagger-jsdoc` update.
- FR-006: The implementation MUST leave explicit deferral notes in the resulting plan or PR description for `camoufox-js`, `express@5`, `jest@30`, and `typescript@6`.
- FR-007: If `playwright-core` validation indicates runtime coupling fallout that requires `camoufox-js` changes, the implementation MUST stop and roll back the `playwright-core` slice instead of widening scope.

### 6.2 Non-Functional Requirements
- NFR-001: The dependency diff MUST stay explainable at direct-package level; unplanned transitive churn is allowed only when directly pulled by an approved package update.
- NFR-002: Validation MUST use concrete repo-local commands and produce evidence that the changed dependency surfaces still work.
- NFR-003: Rollback MUST be possible by restoring `package.json`, `package-lock.json`, and generated artifacts to their prior state.
- NFR-004: The maintenance slice MUST remain planning-safe even if local user modifications exist in `package-lock.json`.

## 7. Success Criteria
- SC-001: `npm audit --json` still reports zero vulnerabilities after the selective update slice.
- SC-002: `npx tsc -p .` passes after the `@types/node` update.
- SC-003: `npm run generate-openapi` completes and `/home/jan/gh/camofox-browser/openapi.json` is regenerated after the `swagger-jsdoc` update.
- SC-004: `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.cjs --runInBand --forceExit tests/unit/openapi.test.js` passes after the `swagger-jsdoc` update.
- SC-005: `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.e2e.cjs --runInBand --forceExit tests/e2e/tabLifecycle.test.js tests/e2e/navigation.test.js tests/e2e/snapshotScreenshot.test.js` passes after the `playwright-core` update.
- SC-006: Post-update review shows that direct dependency changes are limited to the in-scope package set and that deferred migrations remain deferred.

## 8. W-Question Coverage Map
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Repo maintainer implementing dependency maintenance; reviewer validating planning readiness | E-001, E-002 | Yes |
| Was | Selective direct dependency maintenance with explicit deferrals; not a broad package refresh | E-002, E-004, E-005 | Yes |
| Wann | After spec approval and before any implementation; stop immediately if validation implies widened migration scope | E-004, E-005 | Yes |
| Wo | `/home/jan/gh/camofox-browser` only; package files, generated OpenAPI artifact, and explicitly listed validation-sensitive files | E-001, E-006 | Yes |
| Wie | Targeted updates in isolated slices with direct command-based validation and rollback | E-003, E-004, E-005 | Yes |
| Womit | npm, repo-local Jest configs, TypeScript compiler, OpenAPI generator, targeted e2e/unit tests | E-001, E-003, E-006 | Yes |
| Wovon | Current package ranges, lockfile state, runtime/browser coupling, and official migration docs | E-001, E-002, E-004, E-007 | Yes |
| Wogegen | Runtime regression, OpenAPI drift, type drift, accidental major migration, lockfile churn beyond scope | E-005, E-006, E-007 | Yes |
| Warum/Wieso/Weshalb | Audit is already clean; maintenance value comes from freshness, but risk comes from coupling and migration breadth | E-002, E-003, E-004, E-007 | Yes |
| Welche | Evidence includes current command output, inspected code surfaces, and official package docs | E-001 through E-007 | Yes |

## 9. Evidence Ledger
| Evidence ID | Source | Inspection Date | Method | Relevant Claim | Limitations |
|---|---|---|---|---|---|
| E-001 | `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/tsconfig.json`, `/home/jan/gh/camofox-browser/jest.config.cjs`, `/home/jan/gh/camofox-browser/jest.config.e2e.cjs` | 2026-07-04 | read | Declared dependency ranges, Node engine, TS surface, and test runner configuration | Local files only; no claim about package behavior beyond declarations |
| E-002 | `npm audit --json` | 2026-07-04 | bash | Audit posture is clean; this is not a vulnerability-response task | Snapshot in current environment only |
| E-003 | `npm audit fix --dry-run --json` | 2026-07-04 | bash | `npm audit fix` is a no-op for this repo state | Dry-run does not prove future package metadata will stay unchanged |
| E-004 | `npm outdated --json` and `npm ls camoufox-js playwright-core swagger-jsdoc better-sqlite3 impit @types/node --depth=1` | 2026-07-04 | bash | Exact direct outdated packages and current runtime coupling | Outdated view reflects current registry state only |
| E-005 | `npm update --dry-run --json` | 2026-07-04 | bash | Broad update would cause significant direct and transitive churn | Dry-run does not guarantee the exact final lockfile, but it is sufficient for scoping risk |
| E-006 | `/home/jan/gh/camofox-browser/server.js`, `/home/jan/gh/camofox-browser/lib/openapi.js`, `/home/jan/gh/camofox-browser/plugin.ts`, `/home/jan/gh/camofox-browser/tests/e2e/*.test.js` | 2026-07-04 | read, grep, glob | Identified the code and test surfaces most likely affected by targeted updates | Static inspection only; runtime validation still required |
| E-007 | `npm view camoufox-js@0.11.1 peerDependencies dependencies --json`, https://expressjs.com/en/guide/migrating-5.html, https://jestjs.io/docs/upgrading-to-jest30, https://playwright.dev/docs/release-notes, https://www.npmjs.com/package/camoufox-js/v/0.11.1 | 2026-07-04 | bash, read | Major-version migrations, runtime-coupling risks, and the `camoufox-js` peer relationship to `playwright-core` are documented upstream | External docs and registry metadata describe upstream changes, not repo-specific outcomes |

## 10. Decision Ledger
| Decision ID | Decision | Alternatives Considered | Why Chosen | Evidence | Revisit Condition |
|---|---|---|---|---|---|
| D-001 | Exclude `npm audit fix` from the implementation strategy | Use `npm audit fix`; use `npm audit fix --force` | Audit is clean and audit-fix dry-run makes zero changes | E-002, E-003 | Revisit only if a future audit report shows vulnerabilities |
| D-002 | Use selective direct-package updates instead of broad `npm update` | Broad `npm update`; hand-edit lockfile | Broad update would import 34 changes and 10 removals beyond the bounded goal | E-004, E-005 | Revisit only if a dedicated repo-wide dependency refresh is explicitly approved |
| D-003 | Keep `@types/node`, `swagger-jsdoc`, and `playwright-core` in scope | Update nothing; include `camoufox-js`; include major migrations | These are the only direct stale packages with bounded change surfaces under current constraints | E-001, E-004, E-006 | Revisit if targeted validation fails or if package registry state changes materially |
| D-004 | Defer `camoufox-js`, `express@5`, `jest@30`, and `typescript@6` to separate work | Fold them into the same maintenance slice | Their risk profile is migration-heavy or tightly runtime-coupled relative to the maintenance goal | E-004, E-006, E-007 | Revisit only with a dedicated migration spec |
| D-005 | Treat `playwright-core` as its own validation gate inside the same spec | Update `playwright-core` with `camoufox-js`; defer `playwright-core` entirely | The current findings support a bounded trial update, but only with browser-smoke rollback protection | E-004, E-006, E-007 | Revisit if smoke validation shows hidden `camoufox-js` coupling |

## 11. Session Evidence
| Session / Handover | Why Relevant | State Extracted | Confidence | Follow-up Check |
|---|---|---|---|---|
| Current 2026-07-04 dependency-audit session | The spec is derived from findings requested by the user in the current conversation | Dependency posture, risk classification, and target update set were corroborated against current files and fresh command output | High | None |

## 12. Assumptions
- A1: The implementation will preserve the current repository-level Node runtime expectation of 22.x or newer.
- A2: The existing local modification in `/home/jan/gh/camofox-browser/package-lock.json` belongs to the user or current working state and must not be overwritten blindly.
- A3: The selected e2e smoke tests are representative enough to catch obvious `playwright-core` runtime fallout for tab creation, navigation, and snapshot/screenshot behavior.
- A4: No additional hidden repo-local spec convention exists beyond the current absence of a `specs/` directory before this artifact was created.

## 13. Key Entities
- **Selective maintenance slice:** a bounded set of direct dependency updates that share a validation strategy and do not imply major migration work.
- **Deferred migration item:** a dependency update intentionally excluded because it requires a larger design or migration decision.
- **Validation surface:** the exact code or test boundary used to prove a dependency update is safe in this repository.

## 14. Design
### 14.1 Overview
The implementation should proceed in three ordered slices:
1. `@types/node` patch update with explicit type-check validation.
2. `swagger-jsdoc` minor update with OpenAPI regeneration and OpenAPI-unit validation.
3. `playwright-core` minor update with targeted e2e browser smoke validation.

Any slice may be rolled back independently. Failure in a later slice must not retroactively widen scope into deferred migration items.

### 14.2 Data Model and State
No persistent application data model changes are expected. The primary mutable state is dependency metadata in `/home/jan/gh/camofox-browser/package.json` and `/home/jan/gh/camofox-browser/package-lock.json`, plus regenerated OpenAPI output in `/home/jan/gh/camofox-browser/openapi.json`.

### 14.3 APIs and Interfaces
- The repository's HTTP API contract must remain stable through the maintenance slice.
- `swagger-jsdoc` updates must not silently drop, rename, or misparse route documentation.
- `playwright-core` updates must preserve the browser launch and page interaction contract currently exercised by the e2e smoke suite.

### 14.4 UX or Operator Flow
Operator flow for implementation:
1. Update one approved dependency slice.
2. Run the slice-specific validation commands.
3. Inspect the resulting dependency diff for accidental widening.
4. Either keep the slice and continue, or roll it back before starting the next slice.

### 14.5 Migration and Rollout
- No production rollout, port, or data migration is part of this spec.
- The maintenance slice should be implemented as small reviewable commits or equivalent isolated steps.
- If `playwright-core` fails smoke validation, revert that slice and keep the prior successful slices intact.
- Deferred items require separate specs rather than silent continuation.

## 15. Alternatives Considered
| Approach | Pros | Cons | Why not chosen |
|---|---|---|---|
| Run `npm audit fix` | Minimal command surface | No effect in current repo state | Not useful because the audit is already clean |
| Run broad `npm update` | Refreshes many packages at once | Large unexplained churn across runtime and transitive packages | Too wide for a planning-safe maintenance slice |
| Update `camoufox-js` together with `playwright-core` | Might align runtime packages in one pass | Pulls pre-1.0 runtime coupling and native transitive updates into scope | Too risky relative to the current maintenance goal |
| Defer `playwright-core` entirely | Lowest runtime risk | Leaves a direct stale runtime dependency untouched | Not chosen because the current findings support an isolated, validated attempt |

## 16. Research Findings
- Express publishes an explicit migration guide for v5, confirming this is deliberate migration work rather than routine dependency drift.
- Jest publishes an explicit v29-to-v30 upgrade guide, confirming a real test-runner migration surface.
- Playwright release notes show ongoing API and runtime changes across recent minors, justifying runtime smoke validation even for minor updates.
- The `camoufox-js` package metadata and npm package page confirm a peer relationship with `playwright-core` and current pre-1.0 package state, reinforcing the decision to defer `camoufox-js`.

## 17. Technical Context
- Language/Version: JavaScript ESM on Node `>=22`; limited TypeScript validation surface via `/home/jan/gh/camofox-browser/plugin.ts`, while `/home/jan/gh/camofox-browser/workers/crash-reporter/index.ts` exists outside the current `tsconfig.json` file list
- Primary Dependencies: `camoufox-js`, `express`, `playwright-core`, `swagger-jsdoc`, `prom-client`
- Storage: npm package metadata and generated OpenAPI artifact; no application data migration in scope
- Testing Stack: Jest with native ESM and dedicated e2e config
- Platform: Linux development environment; package metadata indicates cross-platform runtime concerns remain important
- Constraints: existing local `package-lock.json` modification, clean audit posture, direct browser-runtime coupling, and generated OpenAPI artifact discipline

## 18. Affected Files
| Absolute Path | Action | Reason |
|---|---|---|
| /home/jan/gh/camofox-browser/package.json | Modify only if targeted install updates direct range metadata | Source of truth for declared dependency intent |
| /home/jan/gh/camofox-browser/package-lock.json | Modify | Carries the actual resolved selective dependency updates |
| /home/jan/gh/camofox-browser/openapi.json | Generate | Must be regenerated after `swagger-jsdoc` update |
| /home/jan/gh/camofox-browser/lib/openapi.js | Validate; modify only if `swagger-jsdoc` behavior drift requires a compatibility change | Primary direct code surface for OpenAPI generation |
| /home/jan/gh/camofox-browser/scripts/generate-openapi.js | Validate; modify only if `swagger-jsdoc` behavior drift requires a compatibility change | Generation entrypoint |
| /home/jan/gh/camofox-browser/tests/unit/openapi.test.js | Validate; modify only if OpenAPI assertions legitimately need compatibility adjustments | Primary OpenAPI regression gate |
| /home/jan/gh/camofox-browser/tsconfig.json | Validate | Defines the type-check surface for `@types/node` update |
| /home/jan/gh/camofox-browser/plugin.ts | Validate | The declared TypeScript compile surface covered by `npx tsc -p .` |
| /home/jan/gh/camofox-browser/server.js | Validate; modify only if `playwright-core` minor drift requires a narrow compatibility change | Runtime browser launch and interaction surface |
| /home/jan/gh/camofox-browser/jest.config.cjs | Validate | Governs targeted unit test execution |
| /home/jan/gh/camofox-browser/jest.config.e2e.cjs | Validate | Governs targeted e2e smoke execution |
| /home/jan/gh/camofox-browser/tests/e2e/globalSetup.js | Validate | Boots shared server/browser state for e2e smoke tests |
| /home/jan/gh/camofox-browser/tests/e2e/tabLifecycle.test.js | Validate | Covers create-tab and lifecycle smoke behavior |
| /home/jan/gh/camofox-browser/tests/e2e/navigation.test.js | Validate | Covers navigation smoke behavior |
| /home/jan/gh/camofox-browser/tests/e2e/snapshotScreenshot.test.js | Validate | Covers snapshot/screenshot smoke behavior |

## 19. Delivery Notes
- DN-001: Keep the implementation split by slice so that each dependency update can be attributed, reviewed, and rolled back independently.
- DN-002: If `package-lock.json` contains unrelated user modifications, implementation must reconcile rather than overwrite them.
- DN-003: The spec intentionally stops at selective maintenance and deferral recording; a separate spec is required before attempting `camoufox-js`, `express@5`, `jest@30`, or `typescript@6`.

## 20. Testing
- [ ] Unit: `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.cjs --runInBand --forceExit tests/unit/openapi.test.js`
- [ ] Integration: `npm run generate-openapi`
- [ ] Type-check: `npx tsc -p .`
- [ ] E2E smoke: `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.e2e.cjs --runInBand --forceExit tests/e2e/tabLifecycle.test.js tests/e2e/navigation.test.js tests/e2e/snapshotScreenshot.test.js`
- [ ] Audit confirmation: `npm audit --json`
- [ ] Diff review: inspect resulting `package.json`, `package-lock.json`, and `openapi.json` changes to confirm scope fidelity

## 21. Risks and Mitigations
- Risk: `playwright-core` minor update exposes hidden coupling with current `camoufox-js`.
- Mitigation: isolate the `playwright-core` slice, run browser-smoke tests, and roll back the slice if the update implies `camoufox-js` work.

- Risk: `swagger-jsdoc` transitive parser changes alter OpenAPI generation.
- Mitigation: regenerate `openapi.json` immediately and gate on `tests/unit/openapi.test.js`.

- Risk: `@types/node` patch surfaces compile-time drift hidden by the current build script.
- Mitigation: run `npx tsc -p .` directly instead of relying on the build script.

- Risk: broad package churn sneaks in via lockfile resolution.
- Mitigation: compare the resulting diff against the allowed package set after each slice.

- Risk: local user changes in `package-lock.json` are overwritten.
- Mitigation: reconcile against current working state before applying dependency updates.

## 22. Rollback Plan
- [ ] Capture the pre-slice state of `/home/jan/gh/camofox-browser/package.json`, `/home/jan/gh/camofox-browser/package-lock.json`, and `/home/jan/gh/camofox-browser/openapi.json` before each slice so local user changes can be restored intentionally.
- [ ] Restore `/home/jan/gh/camofox-browser/package.json` to its pre-slice state if the current slice changed direct ranges unexpectedly.
- [ ] Restore `/home/jan/gh/camofox-browser/package-lock.json` to its pre-slice state for any failed slice.
- [ ] Restore `/home/jan/gh/camofox-browser/openapi.json` to its pre-slice state if `swagger-jsdoc` validation fails.
- [ ] Stop after the failing slice and do not continue into deferred migrations under this spec.

## 23. Open Questions
None

## 24. References
- `/home/jan/gh/camofox-browser/package.json`
- `/home/jan/gh/camofox-browser/tsconfig.json`
- `/home/jan/gh/camofox-browser/jest.config.cjs`
- `/home/jan/gh/camofox-browser/jest.config.e2e.cjs`
- `/home/jan/gh/camofox-browser/server.js`
- `/home/jan/gh/camofox-browser/lib/openapi.js`
- `/home/jan/gh/camofox-browser/plugin.ts`
- `/home/jan/gh/camofox-browser/tests/e2e/tabLifecycle.test.js`
- `/home/jan/gh/camofox-browser/tests/e2e/navigation.test.js`
- `/home/jan/gh/camofox-browser/tests/e2e/snapshotScreenshot.test.js`
- https://expressjs.com/en/guide/migrating-5.html
- https://jestjs.io/docs/upgrading-to-jest30
- https://playwright.dev/docs/release-notes
- https://www.npmjs.com/package/camoufox-js/v/0.11.1
