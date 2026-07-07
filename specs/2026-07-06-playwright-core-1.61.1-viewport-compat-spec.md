# playwright-core 1.61.1 + camoufox-js Viewport Compatibility — Full Spec

**Date:** 2026-07-06
**Project:** camofox-browser
**Status:** Approved
<!-- Approved after second-pass self-review on 2026-07-06. -->

## Spec Review Status
- Review Status: Approved
- Reviewer Type: Second-pass self-review
- Review Date: 2026-07-06
- Blocking Issues Fixed: 0
- Review Artifact: inline approval notes below
- Approval Notes: Second-pass self-review approved the spec as planning-ready. All 24 template sections present, no TODOs/TBDs. W-question coverage complete with evidence references. Affected files individually listed (14 files). Rollback plan concrete (4 steps). A planner can derive exact code changes (2-3 newContext call sites in server.js), validation steps (6 gates), and rollback boundaries without hidden chat context. Non-blocking: A2 (viewport:null prevents Browser.setDefaultViewport) should be empirically validated during implementation — already covered by E2E smoke gate SC-001.

## Review Contract
- **Review goal:** approve a planning-ready spec that unblocks the `playwright-core` 1.61.1 update by working around the `isMobile` protocol incompatibility in Camoufox's bundled Juggler.
- **Review scope:** `server.js` context creation call sites, `playwright-core` dependency update, E2E smoke validation.
- **Review non-scope:** patching the Camoufox Juggler binary, updating `camoufox-js`, other dependency updates, feature changes.
- **Success criteria:** a planner can derive exact code changes, validation steps, and rollback boundaries without reading chat history.
- **Risk classes:** runtime browser regression, snapshot/screenshot behavior change from `viewport: null` pattern, future Playwright protocol additions.
- **Primary evidence sources:** local repo files, E2E test results, `daijro/camoufox#653`, Playwright 1.61 release notes.

## 1. Context and Scope

The predecessor spec (`specs/2026-07-04-selective-dependency-maintenance-spec.md`) deferred the `playwright-core` 1.61.1 update per FR-007 because E2E smoke tests failed with a `Browser.setDefaultViewport` protocol error. The error: Playwright 1.61 sends `isMobile: false` in the viewport payload, but Camoufox's bundled Juggler schema (`pageTypes.Viewport`) only accepts `viewportSize` and `deviceScaleFactor`.

Upstream issue `daijro/camoufox#653` (2026-06-29) confirms this is a known incompatibility. The camoufox Python library pinned `playwright < 1.61` as a workaround. No Juggler patch has been released.

This spec defines a local workaround in `server.js` that avoids the `Browser.setDefaultViewport` call during context creation, enabling `playwright-core` 1.61.1 without upstream camoufox-js changes.

### Current state
- `playwright-core` is at `^1.58.0` (resolved 1.59.1) in `package.json`
- `camoufox-js` remains at `>=0.10.0` (resolved 0.10.2) — unchanged by this spec
- E2E smoke (22 tests) passes on 1.59.1, fails on 1.61.1

## 2. User Scenarios and Testing

### 2.1 User Story 1 — Update playwright-core to 1.61.1 without camoufox-js changes (Priority: P1)

A maintainer wants to update `playwright-core` to the latest version to stay current with Playwright features and bug fixes, without waiting for an upstream camoufox-js Juggler patch.

**Why this priority:** The predecessor spec explicitly deferred this update. Without it, `playwright-core` remains stale indefinitely with no clear upstream timeline for a Juggler fix.

**Independent Test:** Apply the workaround in `server.js`, update `playwright-core`, run E2E smoke tests.

**Acceptance Scenarios:**
1. **Given** `playwright-core` 1.61.1 installed and the `viewport: null` + `setViewportSize()` workaround applied in `server.js`, **When** the maintainer runs E2E smoke tests, **Then** all 22 tests pass (tabLifecycle, navigation, snapshotScreenshot).
2. **Given** the workaround is applied, **When** the maintainer inspects the `server.js` diff, **Then** only the two context-creation call sites changed and no other runtime behavior was modified.
3. **Given** `playwright-core` 1.61.1, **When** `npm audit --json` runs, **Then** no new vulnerabilities are introduced relative to the committed lockfile baseline.

## 3. Edge Cases
- EC1: `page.setViewportSize()` after `viewport: null` context creation may behave differently from context-level `viewport` for snapshot/screenshot dimensions — must be validated by E2E.
- EC2: The health-check context creation (line 709) uses `viewport` + `permissions: ['geolocation']` — the workaround must preserve the `permissions` option.
- EC3: The `session:creating` mutating hook (line 1200) may inject `contextOptions.storageState` — the workaround must not break plugin-injected context options.
- EC4: Future Playwright versions (1.62+) may add more protocol fields the Juggler doesn't accept — this workaround reduces but does not eliminate that risk.
- EC5: The test context at line 5968 uses `browser.newContext()` with no explicit viewport — Playwright may still send a default viewport with `isMobile`. This call site may also need the workaround.

## 4. Goals
- G1: Update `playwright-core` from 1.59.1 to 1.61.1.
- G2: Work around the `isMobile` protocol incompatibility in `server.js` without modifying `camoufox-js`.
- G3: Preserve all existing E2E smoke test behavior (22/22 pass).
- G4: Keep the workaround minimal and isolated to context-creation call sites.

## 5. Non-Goals
- NG1: Do not patch the Camoufox Juggler binary or `camoufox-js` source.
- NG2: Do not update `camoufox-js` from `>=0.10.0` to `^0.11.1` (separate migration spec).
- NG3: Do not update `swagger-jsdoc`, `express`, `jest`, or `typescript` (separate specs).
- NG4: Do not add new tests beyond the existing E2E smoke gates.
- NG5: Do not refactor the context creation pattern beyond the minimal workaround.

## 6. Requirements

### 6.1 Functional Requirements
- FR-001: The implementation MUST update `playwright-core` to `1.61.1` in `package.json` and `package-lock.json`.
- FR-002: The implementation MUST modify both `newContext()` call sites in `server.js` (lines 709 and 1179-1201) to avoid passing `viewport` at context creation time.
- FR-003: The implementation MUST call `page.setViewportSize({ width: 1280, height: 720 })` after `context.newPage()` to restore the intended viewport dimensions.
- FR-004: The implementation MUST preserve the `permissions: ['geolocation']` option in context creation.
- FR-005: The implementation MUST preserve the `session:creating` mutating hook behavior — plugin-injected `contextOptions` must still work.
- FR-006: The implementation MUST evaluate whether the test context at line 5968 also needs the workaround and apply it if necessary.
- FR-007: The implementation MUST NOT modify `camoufox-js`, the Juggler binary, or any `camoufox-js` source file.

### 6.2 Non-Functional Requirements
- NFR-001: The workaround MUST NOT change the user-facing viewport behavior (1280×720 default) observable through the API.
- NFR-002: The workaround MUST NOT introduce measurable latency in context/page creation beyond the additional `setViewportSize` call.
- NFR-003: The `playwright-core` update MUST NOT introduce new npm audit vulnerabilities relative to the committed lockfile baseline.

## 7. Success Criteria
- SC-001: `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.e2e.cjs --runInBand --forceExit tests/e2e/tabLifecycle.test.js tests/e2e/navigation.test.js tests/e2e/snapshotScreenshot.test.js` passes with 22/22 tests.
- SC-002: `npx tsc -p .` passes with zero exit code.
- SC-003: `npm run generate-openapi` succeeds and `openapi.json` is unchanged (no route changes).
- SC-004: `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.cjs --runInBand --forceExit tests/unit/openapi.test.js` passes with 16/16 tests.
- SC-005: `npm audit --json` introduces no new vulnerabilities relative to the committed lockfile baseline (pre-existing vulnerabilities in the committed lockfile are out of scope).
- SC-006: The `server.js` diff is limited to the two (or three if EC5 applies) context-creation call sites.

## 8. W-Question Coverage Map
| W-Question | Answer | Evidence | Blocking if missing? |
|---|---|---|---|
| Wer | Maintainer applying the workaround; reviewer validating E2E smoke | E-001, E-005 | Yes |
| Was | Update playwright-core to 1.61.1 with viewport workaround in server.js | E-001, E-003, E-005 | Yes |
| Wann | After spec approval; stop if E2E smoke fails after workaround | E-005 | Yes |
| Wo | `/home/jan/gh/camofox-browser/server.js` lines 709, 1179-1201, (5968 if EC5 applies); `package.json`, `package-lock.json` | E-001 | Yes |
| Wie | Replace context-level `viewport` with `viewport: null` + post-creation `page.setViewportSize()` | E-002, E-003 | Yes |
| Womit | npm, repo-local Jest configs, TypeScript compiler | E-001, E-005 | Yes |
| Wovon | Predecessor spec findings, upstream issue #653, Playwright 1.61 release notes | E-003, E-004 | Yes |
| Wogegen | Runtime browser regression, snapshot/screenshot behavior change, future protocol additions | E-003, E-005 | Yes |
| Warum/Wieso/Weshalb | Upstream camoufox-js has no Juggler patch timeline; local workaround unblocks the update immediately | E-003 | Yes |
| Welche | E2E test results, upstream issue, release notes, repro scripts | E-002 through E-006 | Yes |

## 9. Evidence Ledger
| Evidence ID | Source | Inspection Date | Method | Relevant Claim | Limitations |
|---|---|---|---|---|---|
| E-001 | `/home/jan/gh/camofox-browser/server.js` lines 709, 1179-1201, 5968 | 2026-07-06 | read, grep | Three `newContext()` call sites; two with explicit `viewport: { width: 1280, height: 720 }`, one with no viewport | Static inspection only |
| E-002 | `.tmp-playwright-repro*.mjs` (3 files in worktree) | 2026-07-06 | read | Repro scripts tested `viewport: null` + `setViewportSize()` as workaround pattern | Not yet validated against full E2E suite |
| E-003 | `daijro/camoufox#653` | 2026-07-06 | web (GitHub issue) | Playwright 1.61 adds `isMobile` to `Browser.setDefaultViewport`; Camoufox Juggler schema rejects it; upstream pinned `playwright < 1.61` | Upstream may release Juggler patch at any time |
| E-004 | Playwright 1.61 release notes (`https://playwright.dev/docs/release-notes`) | 2026-07-06 | web | Firefox 151.0, WebAuthn passkeys, Web Storage; no explicit mention of `isMobile` protocol change in release notes | Release notes don't document protocol-level changes explicitly |
| E-005 | E2E smoke test run in worktree | 2026-07-06 | bash | 20/22 tests failed on 1.61.1 with `Browser.setDefaultViewport` protocol error; 22/22 pass on 1.59.1 | Specific to camoufox-js 0.10.2 + playwright-core 1.61.1 |
| E-006 | 3 parallel worktree repro scripts (`scripts/playwright-camoufox-compat-repro.mjs`, `scripts/camofox-runtime-repro.mjs`) | 2026-07-06 | read | All test same pattern: `newContext()` default vs `viewport: null` + `setViewportSize()` | Repro scripts are exploratory, not formal tests |

## 10. Decision Ledger
| Decision ID | Decision | Alternatives Considered | Why Chosen | Evidence | Revisit Condition |
|---|---|---|---|---|---|
| D-001 | Use `viewport: null` + `page.setViewportSize()` workaround in server.js | Pin `playwright-core < 1.61`; wait for upstream camoufox-js fix | Immediate fix without upstream dependency; `viewport: null` is a valid Playwright pattern | E-003, E-005 | Revisit when upstream camoufox-js releases Juggler patch |
| D-002 | Keep `camoufox-js` at `>=0.10.0` | Update `camoufox-js` to `^0.11.1` | The workaround is in server.js, not camoufox-js; camoufox-js update is a separate migration | E-001, E-003 | Separate spec for camoufox-js migration |
| D-003 | Out-of-scope: Juggler binary patch | Patch `additions/juggler/protocol/Protocol.js` in camoufox source | That's upstream camoufox work, not camofox-browser | E-003 | Revisit if camoufox project accepts PR and releases |

## 11. Session Evidence
| Session / Handover | Why Relevant | State Extracted | Confidence | Follow-up Check |
|---|---|---|---|---|
| 2026-07-04 selective-dependency-maintenance session | Predecessor spec deferred playwright-core 1.61.1 per FR-007 | E2E smoke failed on 1.61.1; rolled back to 1.59.1; committed as `bbb7b44` | High | None — predecessor slice is complete |
| 2026-07-06 framing session | Consolidated context artifact at `local://playwright-core-1.61.1-consolidated-context.md` | Root cause, fix options, scope boundaries | High | None |

## 12. Assumptions
- A1: `page.setViewportSize({ width: 1280, height: 720 })` produces equivalent viewport behavior to context-level `viewport: { width: 1280, height: 720 }` for snapshot/screenshot purposes.
- A2: The `viewport: null` (or `noDefaultViewport: true`) option prevents Playwright from sending `Browser.setDefaultViewport` during context creation.
- A3: The test context at line 5968 may also fail if Playwright sends a default viewport — this will be determined during implementation.
- A4: No other `newContext()` call sites exist in the codebase beyond those identified in `server.js`.

## 13. Key Entities
- **Context-level viewport:** The `viewport` option passed to `browser.newContext()`, which triggers `Browser.setDefaultViewport` in the Juggler protocol.
- **Page-level viewport:** The viewport set via `page.setViewportSize()`, which uses a different protocol command and does not trigger the `isMobile` incompatibility.
- **Juggler protocol schema:** The protocol definition in Camoufox's patched Firefox that defines which properties are accepted; currently missing `isMobile` in `pageTypes.Viewport`.

## 14. Design

### 14.1 Overview
Replace context-level `viewport` with `viewport: null` in `newContext()` calls, then call `page.setViewportSize({ width: 1280, height: 720 })` after `context.newPage()`. This avoids the `Browser.setDefaultViewport` protocol command that carries the unsupported `isMobile` field.

### 14.2 Data Model and State
No data model changes. The viewport dimensions remain 1280×720; only the timing of when they are applied changes (post-page-creation instead of context-creation).

### 14.3 APIs and Interfaces
No HTTP API changes. The `/tabs/:tabId/viewport` endpoint (line 3735) is unaffected — it already uses `page.setViewportSize()`.

### 14.4 UX or Operator Flow
No operator-facing changes. The viewport behavior is identical from the API consumer's perspective.

### 14.5 Migration and Rollout
No migration or phased rollout. The change is a single atomic code modification + dependency update.

## 15. Alternatives Considered
| Approach | Pros | Cons | Why not chosen |
|---|---|---|---|
| Pin `playwright-core < 1.61` | Zero code changes | Leaves dependency stale indefinitely; no clear upstream timeline | Does not solve the problem |
| Wait for upstream camoufox-js Juggler patch | Correct fix at protocol boundary | Indefinite timeline; upstream only pinned Python so far | Unacceptable delay |
| Patch Juggler in camoufox-js source | Correct fix, upstream-able | Requires camoufox-js source modification; separate project; release cycle | Out of scope for camofox-browser |
| `viewport: null` + `setViewportSize()` workaround | Immediate fix; valid Playwright pattern; no upstream dependency | Changes context creation pattern; needs E2E validation | **Chosen** — best risk/value tradeoff |

## 16. Research Findings
- `daijro/camoufox#653` confirms the root cause and that the camoufox maintainer pinned `playwright < 1.61` in the Python library.
- Playwright 1.61 release notes do not explicitly mention the `isMobile` protocol addition, but the issue reporter traced it to the `Browser.setDefaultViewport` command.
- The minimal Juggler patch (adding `isMobile: t.Optional(t.Boolean)` to `pageTypes.Viewport`) was verified working by the issue reporter, confirming the problem is schema validation, not handler logic.

## 17. Technical Context
- Language/Version: JavaScript ESM on Node `>=22`
- Primary Dependencies: `camoufox-js@0.10.2`, `playwright-core@1.59.1` (target: 1.61.1), `express@4`, `swagger-jsdoc@6.2.8`
- Storage: npm package metadata; no application data changes
- Testing Stack: Jest with native ESM, `jest.config.e2e.cjs` for browser smoke tests
- Platform: Linux development environment; Camoufox headless browser
- Constraints: `camoufox-js` pre-1.0 with deep import from `camoufox-js/dist/virtdisplay.js`; build script suppresses TS errors

## 18. Affected Files
| Absolute Path | Action | Reason |
|---|---|---|
| /home/jan/gh/camofox-browser/server.js | Modify | Change `newContext()` call sites to use `viewport: null` + `page.setViewportSize()` workaround |
| /home/jan/gh/camofox-browser/package.json | Modify | Update `playwright-core` range to `^1.61.1` |
| /home/jan/gh/camofox-browser/package-lock.json | Modify | Resolved `playwright-core` 1.61.1 |
| /home/jan/gh/camofox-browser/tsconfig.json | Validate | Type-check surface |
| /home/jan/gh/camofox-browser/plugin.ts | Validate | TypeScript compile target |
| /home/jan/gh/camofox-browser/jest.config.e2e.cjs | Validate | E2E config for smoke tests |
| /home/jan/gh/camofox-browser/tests/e2e/globalSetup.js | Validate | E2e harness bootstrap |
| /home/jan/gh/camofox-browser/tests/e2e/tabLifecycle.test.js | Validate | Browser lifecycle smoke |
| /home/jan/gh/camofox-browser/tests/e2e/navigation.test.js | Validate | Navigation smoke |
| /home/jan/gh/camofox-browser/tests/e2e/snapshotScreenshot.test.js | Validate | Snapshot/screenshot smoke |
| /home/jan/gh/camofox-browser/jest.config.cjs | Validate | Unit test config |
| /home/jan/gh/camofox-browser/tests/unit/openapi.test.js | Validate | OpenAPI regression gate |
| /home/jan/gh/camofox-browser/lib/openapi.js | Validate | OpenAPI generation surface |
| /home/jan/gh/camofox-browser/openapi.json | Validate | Confirm no route changes |

## 19. Delivery Notes
- DN-001: The workaround changes the timing of viewport application (post-page-creation vs context-creation). The planner should ensure the E2E smoke tests validate snapshot/screenshot dimensions, not just navigation.
- DN-002: If EC5 applies (test context at line 5968 also fails), the planner should include that call site in the same slice.
- DN-003: The 3 parallel worktrees (`camofox-modernization`, `camofox-runtime-remediation`, `playwright-camoufox-compat`) explored this same workaround — their repro scripts should be cleaned up after this spec is implemented.

## 20. Testing
- [ ] Type-check: `npx tsc -p .`
- [ ] OpenAPI unit: `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.cjs --runInBand --forceExit tests/unit/openapi.test.js`
- [ ] OpenAPI generation: `npm run generate-openapi`
- [ ] E2E smoke: `env CI= NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.e2e.cjs --runInBand --forceExit tests/e2e/tabLifecycle.test.js tests/e2e/navigation.test.js tests/e2e/snapshotScreenshot.test.js`
- [ ] Audit: `npm audit --json` (compare to committed lockfile baseline)
- [ ] Diff review: inspect `server.js` diff to confirm only context-creation call sites changed

## 21. Risks and Mitigations
- Risk: `page.setViewportSize()` produces different snapshot/screenshot dimensions than context-level `viewport`.
- Mitigation: E2E smoke includes `snapshotScreenshot.test.js` which validates snapshot/screenshot behavior. If it fails, investigate whether `setViewportSize` needs to be called before or after navigation.

- Risk: Future Playwright versions (1.62+) add more protocol fields the Juggler doesn't accept.
- Mitigation: The `viewport: null` workaround avoids the specific `Browser.setDefaultViewport` command, but other protocol commands may also be affected. Monitor upstream camoufox-js for Juggler patches.

- Risk: The `session:creating` mutating hook injects `contextOptions` that include `viewport`.
- Mitigation: The implementation must handle the case where a plugin injects `viewport` into `contextOptions` — either strip it or move it to a post-page-creation `setViewportSize` call.

- Risk: The test context at line 5968 also fails with 1.61.1.
- Mitigation: FR-006 requires evaluating this call site during implementation.

## 22. Rollback Plan
- [ ] Revert `/home/jan/gh/camofox-browser/server.js` to pre-slice state (restore context-level `viewport`)
- [ ] Revert `/home/jan/gh/camofox-browser/package.json` to `playwright-core: ^1.58.0`
- [ ] Revert `/home/jan/gh/camofox-browser/package-lock.json` to pre-slice state
- [ ] Run E2E smoke to confirm rollback restores 22/22 pass on 1.59.1

## 23. Open Questions
- [ ] Should `playwright-core` be pinned to `>=1.61.0 <1.62.0` instead of `^1.61.0` to guard against future protocol additions? Non-blocking — can be decided during planning.

## 24. References
- `/home/jan/gh/camofox-browser/specs/2026-07-04-selective-dependency-maintenance-spec.md` (predecessor spec)
- `/home/jan/gh/camofox-browser/plans/2026-07-04-selective-dependency-maintenance-implementation-plan.md` (predecessor plan)
- `https://github.com/daijro/camoufox/issues/653` (upstream incompatibility issue)
- `https://playwright.dev/docs/release-notes` (Playwright 1.61 release notes)
- `/home/jan/gh/camofox-browser/server.js` (affected source)
- `/home/jan/gh/camofox-browser/AGENTS.md` (project conventions)