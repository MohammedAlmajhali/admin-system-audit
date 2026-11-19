---
name: admin-system-audit
description: Audit an existing admin dashboard against the full system architecture, business workflows, API capabilities, roles and permissions, information architecture, usability, and functional correctness. Use for diagnosing confusing or incomplete administration interfaces.
---

# Admin System Audit

Audit an admin dashboard against what the system actually does, then report before touching code.

## Operating rules (apply in every phase)

1. **Report first, code later.** Phases 1-5 never modify application code, configuration, dependencies, or migrations. Deliverables go under `docs/admin-audit/`. Running tests and the browser may create disposable by-products (temp files, test artifacts, throwaway test scripts, seeded test data); keep these outside tracked application directories (OS temp dir, a git-ignored folder, or `docs/admin-audit/evidence/`), delete what is not evidence, and list any left behind in `SYSTEM_MAP.md`. Tests or scenarios that change data run only against an isolated test environment, never production. Phase 6 starts only after explicit approval.
2. **Evidence or it did not happen.** Every finding cites file paths (with line numbers where possible), a command and its real output, or a screenshot path. No evidence means the finding is marked `Uncertain`.
3. **Never fabricate.** Do not invent endpoints, roles, files, test results, or UI behavior. If a test was not run, write `NOT RUN: <reason>`. If something could not be inspected, write `NOT INSPECTED: <reason>`.
4. **Backend is the source of truth** for capabilities and permissions. Frontend guards are evidence of intent, not enforcement.
5. **Not every endpoint belongs in the admin dashboard.** Parity with the API is not a goal. Never recommend exposing a capability only to achieve feature parity.
6. **Distinguish intentional from defective.** A restriction, a missing screen, or a hidden control may be deliberate. Look for docs, comments, tests, commit history, or permission design that explain it before calling it a bug. If unclear, mark `Uncertain` and list it as a question for the owner.
7. **Be proportionate.** Prefer a smaller number of well-evidenced findings over a long list of speculative ones.

## Confidence and severity vocabulary

Confidence: `Confirmed` (reproduced or directly visible in code/tests) | `Likely` (strong code evidence, not exercised) | `Uncertain` (inference, missing context, or needs owner input).

Severity:
- `Critical`: security/authorization hole, data loss or corruption, admin cannot perform a core duty.
- `High`: a business workflow is blocked, wrong, or misleading; broken contract between UI and API.
- `Medium`: significant friction, confusion, or inconsistency; workaround exists.
- `Low`: polish, minor inconsistency, cosmetic.

Coverage status (traceability matrix): `Complete` | `Partial` | `Missing` | `Unreachable` (screen exists, no route/nav path) | `Dead control` (UI element does nothing or errors) | `Intentionally restricted` | `Out of admin scope` | `Unknown`.

## Large projects: batches and progress tracking

Large systems can exhaust the context window before the reports are done. Work in batches so nothing is lost or repeated:

1. **Resume check first.** If `docs/admin-audit/PROGRESS.md` exists, read it and continue from it. Do not redo modules marked `Done` unless the user asks or the code changed.
2. **Define modules early.** After a quick scan, split the system into modules (for example by business domain or navigation section: users, orders, billing, content, settings) and list them in `PROGRESS.md` with a status per phase: `Not started | In progress | Done | Blocked`.
3. **Audit module by module.** For each module run discovery, coverage, UX, and quality checks, and write its findings into the relevant report sections before moving to the next module. Do not hold findings only in memory.
4. **Keep `PROGRESS.md` current**, after every module: modules and statuses, what was inspected, what was not (`NOT INSPECTED: reason`), tests and scenarios run, open owner questions, and the next step. It is a supporting file, not one of the five reports.
5. **Cross-cutting passes last**: navigation tree, glossary, permission consistency, and the blueprint are assembled after the per-module work, from what the reports already contain.
6. For small projects, skip the module split and keep a single-line status in `PROGRESS.md`.

## PHASE 1: System discovery

Goal: understand what the system really does before judging the dashboard.

Inspect the whole relevant repository, not just the admin frontend:

- **Stack and layout**: languages, frameworks, monorepo structure, how the admin app is built, served, and authenticated. Read README, docs, ADRs, CHANGELOG, `.env.example`, CI config.
- **Backend**: models/entities, relations, enums and state machines, serializers/schemas, API endpoints (method, path, handler), services/use-cases, background jobs, signals/events, webhooks, management commands.
- **Roles and permissions**: role definitions, permission classes/decorators/middleware/policies, object-level checks, tenant or organization scoping, superuser behavior, frontend route guards and conditional rendering. Record where frontend and backend disagree.
- **Frontend**: route table, navigation config, layouts, pages, shared components, API client layer, state management, i18n files, feature flags.
- **Workflows**: trace real business processes end to end (for example create, review, approve, fulfill, refund, suspend), including status transitions and who may trigger each.
- **Business intent**: for each capability, work out why it exists (who needs it, what business outcome it serves, what goes wrong if it fails) from code, docs, tests, naming, and history. Code shows what the system does, not always why.
- **Tests and docs**: existing unit, integration, e2e tests and what they cover; any product or API documentation.

Then classify every backend endpoint/operation:

| Class | Meaning |
|---|---|
| Admin-relevant | Operators or admins need to perform or oversee it |
| End-user only | Belongs to customer/member-facing product |
| System-internal | Jobs, service-to-service, health checks |
| Public / webhook | Unauthenticated or third-party callbacks |
| Deprecated / unclear | Appears unused or superseded |

Use these questions to decide admin relevance: Does a business role need to do or oversee this? Is it support, operations, configuration, moderation, finance, or compliance work? Would exposing it add risk (destructive, financial, privacy-sensitive, impersonation, secrets)? Record the reasoning, not just the label.

### Business intent and questions for the owner

Do not assume business rules. For each core capability record: (a) the inferred purpose, (b) the evidence for it, (c) what you could not determine. When a rule is unclear and the answer would change whether something is a defect, an intentional restriction, or a missing feature, add it to a **Questions for owner** list. Each question must be specific, answerable, and cite the code that prompted it.

- Ask the user once, in a single batched message, at the end of Phase 2 (before finalizing the reports) when answers would change the classification of several findings or of any Critical/High item.
- Continue with every part of the audit that does not depend on the answers. If no answer arrives, proceed, mark the affected findings `Uncertain`, and keep the questions listed in `SYSTEM_MAP.md`.

Output of this phase feeds `SYSTEM_MAP.md`.

## PHASE 2: Feature coverage audit

Build a traceability matrix. One row per business capability (not per endpoint):

| Capability | Backend support | Permitted role(s) | Dashboard page/component | Action | Status | Evidence | Notes |
|---|---|---|---|---|---|---|---|

Rules:
- Start from business entities and workflows from Phase 1, then map to backend, then to UI. Also scan the other direction: every dashboard page/control must trace back to a capability and an endpoint; flag orphans.
- For each entity, check CRUD and lifecycle actions separately (list, view, create, edit, delete/archive, restore, bulk actions, export, search, filter, sort, pagination, status transitions). Mark which are intended.
- Identify:
  - truly missing management functionality (capability needed by a role, supported by backend, absent in UI)
  - unreachable screens (route or component exists, no navigation or link path, or hidden by a wrong guard)
  - dead controls (buttons, menu items, toggles, tabs that do nothing, call missing endpoints, or always error)
  - incomplete CRUD (for example create exists but no edit, or delete without confirmation or dependency handling)
  - permission inconsistencies (UI shows an action the role cannot perform, UI hides an action the role can perform, backend enforces less than the UI implies)
- For each gap, state whether it is a **defect**, an **intentional restriction**, or **uncertain**, and why.
- Never list a protected capability as a gap solely because the backend has it.

### Priority inputs: task frequency and business criticality

For each capability, record how often staff perform it and how critical it is, so priorities follow real usage and not guesses:
- `Task frequency`: `Daily | Weekly | Occasional | Rare | Unknown`
- `Business criticality`: `Critical | Important | Minor | Unknown`
- Source for each value: usage analytics, access or audit logs, support tickets, documentation, or the owner's answer. If no data exists, write `Unknown` and add it to Questions for owner. Never infer frequency or criticality from how the code looks.

These values feed impact scoring in `IMPLEMENTATION_PLAN.md`.

Output feeds `FEATURE_COVERAGE.md`.

## PHASE 3: UX and information architecture audit

Review the whole product surface, not isolated screens.

- **Navigation tree**: extract the full tree (sidebar, top bar, tabs, breadcrumbs, command palette, in-page links). Note depth, grouping logic, ordering, and per-role differences.
- **Labels and terminology**: duplicate or ambiguous labels, same concept with different names (or different concepts with the same name), mismatches between UI wording and backend/domain terms, mixed languages, inconsistent casing and tense.
- **Placement and redundancy**: misplaced items, screens that overlap, items that belong under a different parent, settings scattered across places.
- **Discoverability**: can each role find each task from where they start? Count clicks for key tasks. Check search, empty-state guidance, contextual help, and tooltips for jargon.
- **Task flows**: walk the top tasks per role end to end (including error recovery). Note dead ends, missing confirmations, missing success feedback, lost context after save, and unclear next steps.
- **Forms**: field order and grouping, required/optional marking, validation timing and message quality, defaults, help text, destructive-action confirmation, unsaved-changes handling.
- **Tables and lists**: column choice, sorting, filtering, pagination, bulk actions, row actions, density, status presentation, export.
- **States**: empty, loading, error, partial-failure, permission-denied, and not-found states for each major screen.
- **Responsive behavior**: layout at common breakpoints, table overflow, touch target size, navigation collapse.
- **Accessibility**: semantic structure, keyboard operation, focus management, labels and ARIA, color contrast, error announcement, reduced motion. Use automated checks where available, and say so when only code review was done.

### Visual inspection (required when the environment permits)

Reading components is not enough. Start the dashboard with the project's documented commands (dev server, docker compose, seed data or fixtures) against a non-production backend, then inspect the rendered UI for each role:
- capture screenshots of every major screen per role, at desktop and mobile widths (and dark mode if supported), saved under `docs/admin-audit/evidence/`
- compare what renders with what the code suggests (hidden items, overflowing tables, truncated labels, missing states, broken layouts)
- if the app cannot be started, write `NOT RUN: <reason>` and downgrade UI findings to `Likely` or `Uncertain`

### Arabic, RTL, and localization audit (when the UI uses Arabic or another RTL language, or ships multiple locales)

- `lang` and `dir` are set correctly per locale, and the layout actually mirrors (sidebar side, breadcrumbs, pagination arrows, steppers, progress bars, drawers, toasts).
- CSS uses logical properties (`margin-inline-start`, `padding-inline-end`, `inset-inline-*`) instead of hardcoded left/right; flag `text-align: left`, absolute left/right offsets, and transforms that do not flip.
- Directional icons (back, next, chevrons, undo) are mirrored; non-directional icons are not.
- Mixed-direction text renders in the right order: Latin names, emails, URLs, phone numbers, IDs, and numbers inside Arabic text, especially in tables and forms (check `bdi` / `unicode-bidi` use).
- Numerals, dates, calendars (Gregorian/Hijri if relevant), and currency format and placement are consistent and match user expectations.
- Arabic terminology is clear and consistent: one Arabic term per concept, no unexplained mixing of English and Arabic for the same concept, no machine-translation artifacts, one consistent register. Add an Arabic column to the glossary.
- Missing translations, hardcoded strings, untranslated backend error messages, concatenated strings that break grammar, and plural forms.
- Fonts with proper Arabic glyph support and line-height, text expansion and truncation, column widths, form label alignment.
- Walk at least one real Arabic-first task flow in the browser, not just the locale files.

Then propose, as the **Dashboard Redesign Blueprint** (full structure under `PROPOSED_STRUCTURE.md` in Phase 5):
- a clearer navigation hierarchy (current tree vs proposed tree, with a mapping table of old to new location per item), including which items to merge, rename, move, remove, or add, where each capability lives, and how users move between pages, and
- a **naming glossary**: `Term | Definition | Variants found (with file paths) | Recommended label`.

Output feeds `UX_ISSUES.md` and `PROPOSED_STRUCTURE.md`.

## PHASE 4: Bug and quality audit

Check, with evidence:

- **Routes**: broken or duplicate routes, wrong redirects, missing 404 handling, guards that disagree with permissions.
- **UI-to-API contracts**: request/response shape mismatches, wrong HTTP methods or paths, ignored fields, renamed fields, pagination/filter/sort parameters the backend does not support, enum drift between frontend and backend.
- **Validation**: client vs server rules that differ, missing server validation, unhelpful error mapping.
- **State handling**: stale caches after mutations, race conditions, optimistic updates without rollback, lost form state, loading flags that never clear.
- **Authorization**: missing server-side checks, object-level access gaps, tenant leakage, privilege escalation paths, sensitive data returned but not needed.
- **Error handling**: swallowed errors, raw error dumps shown to users, missing retry or recovery paths.
- **Workflows**: invalid state transitions allowed, partial failures leaving inconsistent data, missing audit trail for sensitive actions.

Execution:
- **Run existing tests.** Detect the commands from `package.json`, `Makefile`, `pyproject.toml`, `pytest.ini`, CI config, or docs. Record the exact command, working directory, exit code, and the real summary (passed/failed/skipped counts and failing test names). If tests cannot run, record why. Do not install global tooling or modify the environment beyond what the project documents without saying so.
- **Browser testing is required whenever the environment permits it** (Playwright, Cypress, or a browser tool; see Visual inspection in Phase 3) and a safe, non-production target exists. Try to start the app yourself from documented commands before concluding it cannot run. Use seeded or test data and test credentials per role. Never test against production and never use real user data. Save screenshots under `docs/admin-audit/evidence/` and reference them. If browser testing is not possible, write `NOT RUN: <reason>` and mark UI findings as `Likely` or `Uncertain` accordingly.
- Test coverage gaps are findings too: note critical workflows with no tests.
- **Test environment safety:** anything that writes, deletes, or sends (data mutations, emails, payments, webhooks) runs only against an isolated test database or sandbox with test credentials and stubbed external services. If no such environment exists, do not run the mutating scenario: mark it `NOT RUN: no isolated test environment` and use a `Static trace`. Temp files follow operating rule 1.

### Workflow simulation

Exercise realistic scenarios end to end instead of judging screens in isolation. For each core entity or workflow and each relevant role, define scenarios and run them in the browser (or through the API when the UI cannot be driven), using seeded or test data only:
- happy path (create, edit, complete)
- reversal paths (cancel, reject, undo, archive and restore)
- follow-up (find it again, track its status, see history or audit entries)
- failure paths (invalid input, conflicting state, a dependency that blocks deletion, API or network error)
- permission paths (a role without access attempts the action through the UI, by direct URL, and by direct API call)

Record each in a table: `Scenario | Role | Steps | Expected | Actual | Result (Pass / Fail / NOT RUN / Static trace) | Evidence`. A result reached only by reading code is labeled `Static trace`, never `Pass`.

Output feeds `UX_ISSUES.md` (UI-visible issues) and `IMPLEMENTATION_PLAN.md` (everything), plus a "Test and verification log" section in `SYSTEM_MAP.md`.

## PHASE 5: Report first

Before changing any application code, create `docs/admin-audit/` with these five mandatory reports. Supporting material is allowed: `docs/admin-audit/PROGRESS.md` and evidence (screenshots, logs, scripts, diagrams) in subfolders such as `docs/admin-audit/evidence/`. Evidence must be referenced from the reports, and nothing supporting replaces or merges any of the five:

- `SYSTEM_MAP.md`
- `FEATURE_COVERAGE.md`
- `UX_ISSUES.md`
- `PROPOSED_STRUCTURE.md`
- `IMPLEMENTATION_PLAN.md`

### Issue format (use for every issue in every file)

```
### <ID> <Short title>
- **Severity:** Critical | High | Medium | Low
- **Confidence:** Confirmed | Likely | Uncertain
- **Category:** Coverage | Permissions | Navigation | Terminology | Form | Table | State | Accessibility | Contract | Validation | Bug | Test gap
- **Evidence:** `path/to/file.ext:LINE` (what it shows) / command + output excerpt / screenshot path
- **User impact:** who is affected, doing what, and what goes wrong
- **Proposed fix:** concrete change, scoped to the smallest sensible unit
- **Risk / effort:** S | M | L, and what could break
- **Open questions:** (required when Confidence is Uncertain)
```

Use stable ID prefixes: `COV-` (coverage), `UX-`, `BUG-`, `SEC-`, `IA-`, `TEST-`.

### File templates

**SYSTEM_MAP.md**
1. Audit inputs: what was available (backend code, frontend code, runnable UI, test accounts per role, usage data, docs) and what was missing, so readers know which findings could not be confirmed
2. Repository overview (stack, structure, how admin is built and served)
3. Business entities and relationships
4. Business workflows and state transitions
5. Roles and permission model (with backend vs frontend discrepancies)
6. API surface classified per Phase 1 table (admin-relevant / end-user only / internal / public / deprecated)
7. Business intent per core capability (purpose, evidence, what could not be determined)
8. Existing tests and docs
9. Test and verification log (command, exit code, real result, or `NOT RUN: reason`), the workflow simulation table, and any temp files left behind
10. Questions for owner (specific, with the code evidence that prompted each; mark which are blocking)
11. Assumptions and unknowns

**FEATURE_COVERAGE.md**
1. Summary counts by status
2. Traceability matrix (columns from Phase 2)
3. Orphan UI elements (no backing capability)
4. Intentional restrictions (with the evidence that shows intent)
5. Capability priority table: `Capability | Role | Task frequency | Business criticality | Source (usage data / owner / Unknown)`
6. Defects and uncertain gaps in the issue format

**UX_ISSUES.md**
1. Current navigation tree (per role where it differs)
2. Findings grouped by: navigation, terminology, task flows, forms, tables, states, responsive, accessibility
3. Naming inconsistencies table (feeds the glossary)
4. Bugs found in Phase 4 that are user-visible, in the issue format

**PROPOSED_STRUCTURE.md** (the Dashboard Redesign Blueprint)
1. Design principles used (short)
2. Disposition of every existing screen and menu item:
   `Item | Current location | Disposition (Keep / Rename / Move / Merge / Split / Remove / Defer) | Target | Reason | Linked issues`
3. Functionality to add:
   `Capability | Role | Proposed location | Screen type (list / detail / form / wizard / dashboard) | Backing endpoints (existing or new) | Protected? | Priority`
4. Proposed navigation tree (per role where it differs), next to the current tree
5. Page-level blueprint for each new or restructured screen: purpose, primary user and task, key content, actions, states (empty / loading / error / denied), entry points, exit points
6. Navigation flows for the top tasks: list to detail to edit and back, cross-links between related entities, breadcrumbs, and where the user lands after each action (a text or Mermaid diagram is fine)
7. Naming glossary (with an Arabic column when the UI is Arabic)
8. Route and redirect plan to keep existing links working
9. Items deliberately **not** added to the dashboard, with the reason (protected, end-user only, internal)
10. Open design decisions that need the owner

**IMPLEMENTATION_PLAN.md**
1. Approval status header: `Status: AWAITING APPROVAL`
2. Task table sorted by **impact (high to low), then risk (low to high)**. Impact takes task frequency and business criticality as inputs (`Unknown` when there is no usage data or owner answer, never guessed):

   `Task ID | Linked issues | Blueprint items | Description | Frequency | Criticality | Impact | Risk | Effort | Depends on | API change? | Tests to add/update | Approval`
3. Module-by-module grouping and suggested order
4. Items requiring a product or security decision before work can start
5. Rollout and rollback notes

### Architecture validation gate

Code does not tell you how staff actually want to run the system. After the reports are written and before asking for implementation approval, present the proposed dashboard to the owner in a short summary: the proposed navigation tree per role, the main merges, moves, removals, and additions, and the naming glossary. Ask the owner to confirm that the grouping and names match how the organization really works.

Record the result in `PROPOSED_STRUCTURE.md` under a section **Owner validation**, with one row per blueprint item: `Item | Status (Confirmed / Changed / Rejected / Pending) | Owner comment`. Revise the blueprint and `IMPLEMENTATION_PLAN.md` for anything `Changed` or `Rejected`. Structural and naming changes may be implemented only when their blueprint items are `Confirmed`.

After writing the reports and running the validation gate, **stop**. Summarize: counts by severity, top 5 issues, uncertain findings needing owner input, anything not run or not inspected, and ask which task IDs are approved. Do not continue to Phase 6 on your own.

## PHASE 6: Implementation after approval

Proceed only when the user explicitly approves, naming tasks or issue IDs (or clearly approving the whole plan). Silence, "looks good" about the reports, or approval of a different scope is not approval to change code. If approval is partial, implement only the approved items. Navigation, naming, and layout changes additionally require their blueprint items to be `Confirmed` in the Owner validation table.

Execution rules:
- Work **module by module** (for example one navigation section or one entity at a time), in the approved order. Prefer a branch and small, reviewable commits. Update the task's approval/status in `IMPLEMENTATION_PLAN.md` as you go.
- **Preserve business logic and API compatibility.** Do not rename, remove, or change the shape of endpoints, payloads, permissions, or database fields unless that change was separately and explicitly approved. Keep old routes working through redirects when navigation changes.
- **Never expose protected capabilities** (destructive operations, impersonation, secrets, payments, privacy-sensitive data, superuser-only actions) just to reach feature parity. If a task seems to require it, stop and ask.
- **Respect the backend permission model.** Frontend guards must mirror backend rules, never replace them.
- **Tests:** add or update tests for each changed behavior (unit and, where practical, integration or e2e for affected workflows). Run the relevant suites after each module and report real results.
- **Verify each affected user workflow** end to end (with browser testing when permitted), for each affected role, including empty, error, and permission-denied states.
- After each module, report: what changed (files), tests run with actual results, workflows verified, and anything left or deferred. Flag any deviation from the plan before continuing.

## Audit-only mode

If the user asks for "audit-only", "report only", or gives no approval, run Phases 1-5, write the reports, and stop. Do not modify application code, configuration, dependencies, or migrations.

## Quick start checklist

- [ ] Confirm the repository root and which app is "the admin dashboard"
- [ ] Check for `docs/admin-audit/PROGRESS.md` and resume from it; for large systems define modules and audit in batches
- [ ] Phase 1: system discovery complete, endpoints classified, business intent recorded, owner questions listed
- [ ] Phase 2: matrix built, intentional vs defect separated, blocking questions asked once
- [ ] Phase 3: nav tree, glossary, task flows reviewed; rendered UI inspected per role (or `NOT RUN`); Arabic/RTL audit done when applicable
- [ ] Phase 4: tests run (real results or `NOT RUN`), workflow scenarios simulated per role (or `NOT RUN` / `Static trace`)
- [ ] Phase 5: all five reports in `docs/admin-audit/` (evidence in subfolders), every issue has evidence, severity, impact, fix, confidence; blueprint complete
- [ ] Priority inputs (frequency, criticality) recorded from data or owner answers, `Unknown` otherwise
- [ ] Architecture validation gate run; Owner validation table filled
- [ ] Stopped and asked for approval before any code change
