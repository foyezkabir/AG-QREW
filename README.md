# AG-QREW - Agentic QREW

> **AG-QREW** stands for **Agentic QREW** - an autonomous QA agent pipeline powered by Claude Code.



A multi-agent QA automation system built for Claude Code. Point it at any business document - Jira story, epic, BRD, PR, release note, or paste content directly - press go, and receive a fully executed test cycle: test cases in TestRail, Playwright scripts that run, bugs in Jira or Confluence, and a signed-off report - all without writing a single test manually.

---

## What it is

AG-QREW is a team of five specialised AI agents that coordinate through a shared task list to execute a complete sprint QA cycle. Each agent owns a specific domain, runs in the background in parallel, and communicates only through structured signals. No agent ever asks the user a question - only the orchestrator (QA Lead) does, and only at three defined points.

**One human decision.** After the test plan is generated, the user types `proceed`. Everything from test case writing to bug reporting to sign-off is fully autonomous from that point.

---

## Architecture

```
                          USER
                            │
                            │  /qa-lead  (start sprint)
                            ▼
                    ┌───────────────┐
                    │   QA Lead     │  ← only agent that talks to user
                    │ (orchestrator)│
                    └──────┬────────┘
                           │ spawns all agents at once (Phase 2)
          ┌────────────────┼────────────────────┐
          │                │                    │
          ▼                ▼                    ▼
  ┌──────────────┐ ┌──────────────┐  ┌──────────────────┐
  │ qa-tc-writer │ │qa-script-    │  │  qa-api-tester   │
  │              │ │writer        │  │                  │
  │ Test cases   │ │ Playwright   │  │ Postman/Newman   │
  │ → TestRail   │ │ automation   │  │ API tests        │
  └──────────────┘ └──────────────┘  └──────────────────┘
          │                │
          │ TC-READY signal│
          └───────┐        │
                  ▼        ▼
           ┌─────────────────┐
           │    qa-hawk      │
           │                 │
           │ Smoke + explore │
           │ → TestRail      │
           │ → Bug report    │
           └─────────────────┘

  All agents coordinate via:  qa/shared-task-list.txt
```

---

## The Five Agents

### QA Lead
**File:** `.claude/commands/qa-lead.md`
**Role:** Orchestrator. The only agent that interacts with the user.

- Fetches sprint documentation from Confluence (or accepts pasted content)
- Runs a Sprint Health Scan against Jira to score risk (LOW / MEDIUM / HIGH)
- Analyses any business document for requirements gaps - Jira story, epic, BRD, PR, release note, or pasted content
- Writes a full SFDIPOT test plan covering all in-scope modules
- Creates the TestRail milestone, project, and suite
- Spawns all agents simultaneously after plan approval
- Monitors the shared task list, consolidates bugs, updates Jira
- Produces the final sign-off report

---

### qa-tc-writer
**File:** `.claude/commands/qa-tc-writer.md`
**Role:** Writes structured test cases and imports them to TestRail.

- Reads the approved test plan and maps features to modules
- Writes plain-text `.txt` test case files per module (`login-tc.txt`, etc.)
- Each file covers: UI, Functional–Positive, Functional–Negative/Boundary, Mobile Responsive
- Runs a 2-round silent self-verification gap check after every module
- Creates nested TestRail sections per module:
  ```
  Module (parent)
    ├── UI
    ├── Functional - Positive
    ├── Functional - Negative / Boundary
    └── Mobile Responsive
  ```
- Imports cases sequentially (500ms pacing, 3 retries, PROGRESS every 5 cases)
- Creates the sprint TestRail run after all modules are imported

**TC ID format:** `TC-001`, `TC-002`… per file. Simple sequential, no module prefix. TestRail assigns its own permanent IDs (C51094…) after import.

```
TC-001
------
Title:          Verify that the user is redirected to the dashboard after login
Type:           Functional
Priority:       High
Preconditions:  User account exists and is active
Steps:
  1. Navigate to /login
  2. Enter 'admin@test.com' in the Email field
  3. Enter valid password in the Password field
  4. Click the Sign In button
Test Data:
  - Email: admin@test.com
  - Password: Test@1234
Expected:
  - Should redirect to /dashboard
  - Should display the user's name in the header
TestRail ID:    [pending]


TC-002
------
Title:          Verify that an error message is shown when the password is incorrect
Type:           Negative
Priority:       High
Preconditions:  User account exists
Steps:
  1. Navigate to /login
  2. Enter 'admin@test.com' in the Email field
  3. Enter 'wrongpassword' in the Password field
  4. Click the Sign In button
Test Data:
  - Email: admin@test.com
  - Password: wrongpassword
Expected:
  - Should display "Invalid email or password" below the form
  - Should remain on the /login page
TestRail ID:    [pending]
```

---

### qa-script-writer
**File:** `.claude/commands/qa-script-writer.md`
**Role:** Writes and executes Playwright end-to-end automation scripts.

Follows a 4-tier architecture:

| Tier | Folder | Contains |
|------|--------|----------|
| 1 | `locators/` | Selectors only - arrow function properties |
| 2 | `pages/` | Interaction methods - click*, fill*, get*, verify* |
| 3 | `datas/` | Static test data + Faker factories |
| 4 | `tests/` | Pure test specs - max 10 lines, no logic |

**Phase 0 - Bootstrap** (before any code is written):
- `npm init`, install `@playwright/test @types/node @faker-js/faker @axe-core/playwright`
- Creates `tsconfig.json`, `playwright.config.ts`, installs Chromium + WebKit
- Verifies `npx tsc --noEmit` reaches zero errors

**Phase 1A - Exploration** (runs parallel to qa-tc-writer):
- Uses `browser_snapshot` (accessibility tree) to extract all interactive elements
- Writes Locators and Page Objects for every module

**Phase 1B - Test Data** (starts per module as `TC-READY` signals arrive):
- Writes `*Data.ts` (static reference values) and `*Factory.ts` (Faker-generated inputs)

**Phase 2 - Specs** (one module at a time from TC files):
- Maps every `TC-NNN` block to a `[TC-NNN] (TR:xxx): Verify that...` test
- Mobile TCs tagged `@mobile` - Playwright config routes them to iPhone 13, Pixel 5, iPad viewports
- Negative/error-state tests use `NetworkHelper` to mock API failures via `page.route()`
- Every `navigate()` runs a WCAG 2.1 AA axe scan and a 3s load time check (both soft - log only)

**Phase 3 - Reconciliation + Quality**:
- Coverage gap fill, TR ID resolution
- Flakiness detection via `--repeat-each=3`
- Generates `.github/workflows/playwright.yml` CI config
- Writes `coverage-matrix.txt` (TC-ID → TR-ID → spec → pass/fail)

---

### qa-hawk
**File:** `.claude/commands/qa-hawk.md`
**Role:** Smoke testing, SFDIPOT exploratory testing, bug logging, TestRail result recording.

Operates in four modes triggered by QA Lead:

| Mode | Trigger | What it does |
|------|---------|--------------|
| 0 - Environment | `HAWK-TASK \| mode: environment` | Validates all URLs, auth, seed data before any testing begins |
| 1 - Smoke | `HAWK-TASK \| mode: smoke` | Runs the core happy path; records Pass/Fail to every TestRail case in the module immediately |
| 2 - Explore | `HAWK-TASK \| mode: explore` | Full SFDIPOT + FEW HICCUPPS oracle testing; posts TestRail result per TC as executed; links each bug to its TestRail case (status=Failed) |
| 3 - Retest | `RETEST: {JIRA-KEY}` | Re-runs reproduction steps after a fix; updates TestRail and Jira |

**Bug report format:** Every bug includes Feature, Severity (with 5-dimensional matrix), Oracle, TC-ID, Title, Steps, Expected, Actual, Screenshot, and Video (if available from qa-script-writer's Playwright report).

---

### qa-api-tester
**File:** `.claude/commands/qa-api-tester.md`
**Role:** Tests all changed API endpoints via Postman and Newman.

- Parses the Swagger/OpenAPI spec for endpoints changed this sprint
- Builds a structured Postman collection with positive, negative, boundary, and SQL injection tests
- Runs the collection via Newman CLI
- Imports the collection and results to the team's Postman workspace
- Verifies enum values against the live API before writing tests

---

## Full Workflow

```
Phase 0 - Setup (QA Lead, blocking)
  ├── .env check - credentials verified or collected from user
  ├── Document intake - Confluence fetch or pasted BRD/release note
  ├── Atlassian auth - MCP authentication check
  ├── Browser preflight - browser_navigate + browser_snapshot on staging URL
  │     if denied → surface to user immediately, do NOT proceed
  ├── API docs check - Swagger link or skip
  └── Sprint Health Scan - Jira signals → risk score (0-10 → LOW/MEDIUM/HIGH)

Phase 1 - Test Plan (QA Lead, blocking)
  ├── Gap analysis on source document (Jira story, epic, BRD, PR, release note, or paste)
  ├── Write qa/test-plan-sprint{N}.txt (SFDIPOT coverage per module)
  ├── Self-verification round on the plan
  └── ▶ Present to user - wait for "proceed"

Phase 2 - Agents spawn (QA Lead, ONE message, all simultaneous)
  ├── qa-hawk        → mode: environment (health check - gates everyone)
  ├── qa-tc-writer   → write test cases
  ├── qa-script-writer → Phase 0 bootstrap + explore site
  └── qa-api-tester  → build and run API tests

── PIPELINE RUNS AUTONOMOUSLY ────────────────────────────────────────────────

qa-tc-writer (per module loop):
  write TCs (TC-001, TC-002...)
    → 2-round silent gap check
    → create TestRail sections (parent + 4 child: UI / Func+ / Func- / Mobile)
    → import sequentially (500ms + 3 retries)
    → PROGRESS every 5 cases
    → MODULE-DONE + TC-READY signals
  after all modules:
    → create TestRail run → META: testrail_run_id written

qa-script-writer (parallel with tc-writer):
  Phase 0:  npm install → tsconfig → playwright.config → tsc: 0 errors
  Phase 1A: browser_snapshot → Locators + Page Objects (all modules)
  Phase 1B: Data + Factory files (per TC-READY signal)
  Phase 2:  spec per module → run → fix errors
  Phase 3:  gap fill → flakiness detection → CI yaml → coverage matrix

qa-api-tester:
  parse spec → plan collection → build in Postman → run Newman → import results

qa-hawk (smoke + explore, triggered per module by QA Lead):
  Smoke:
    read case_ids from SECTION-DONE signal
    run core happy path
    POST Pass/Fail to every case in TestRail immediately
  Explore:
    write session charter (SFDIPOT mandate)
    execute every TC block step-by-step
    POST result to TestRail per case as executed
    SFDIPOT dimensions + FEW HICCUPPS oracles
    log bugs → POST Failed to matching TestRail case immediately

Phase 3 - Consolidation (QA Lead, silent)
  ├── Read DONE signals from all agents
  ├── Validate bug report format
  ├── Push bugs to Jira / Confluence
  └── Trigger retests on fixed issues

Phase 4 - Sign-off (QA Lead → user)
  ├── Decision gate record (pass rate, bugs, coverage, flakiness)
  ├── AUTOMATION QUALITY section (flaky list, perf flags, a11y summary)
  └── ▶ Present sign-off report - PASS / CONDITIONAL / FAIL
```

---

## What Gets Generated

```
qa/
  test-plan-sprint{N}.txt         Full SFDIPOT test plan
  shared-task-list.txt            Agent coordination bus (all signals)
  hawk-env-sprint{N}.txt          Environment validation report
  test-cases/
    {module}-tc.txt               Structured test cases per module
  automation/
    locators/    {Module}Locators.ts
    pages/       {Module}Page.ts
    datas/       {Module}Data.ts + {Module}Factory.ts
    tests/       {module}.spec.ts
    fixtures/    base.ts
    helpers/     NetworkHelper.ts + LoopHelper + ConditionalHelper + ...
    utils/       perf.ts + reportPath.ts + globalTeardown.ts
    auth.json
    playwright.config.ts
    tsconfig.json
    package.json
    playwright-report/            HTML report + videos (on failure)
    coverage-matrix.txt           TC-ID → TR-ID → spec → pass/fail
  api-tests/                      Newman results + Postman collection export
  bugs/
    bug-report-sprint{N}.txt      All bugs (SFDIPOT + FEW HICCUPPS oracles)
    screenshots/                  bug-001.png, bug-002.png...
  reports/
    sign-off-sprint{N}.txt        Final QA sign-off report

.github/
  workflows/
    playwright.yml                CI config (Chromium + WebKit, artifact upload)
```

---

## Signal Protocol

Agents coordinate exclusively through `qa/shared-task-list.txt`. No direct agent-to-agent calls.

| Signal | Written by | Read by | Meaning |
|--------|-----------|---------|---------|
| `TC-READY: {module}` | qa-tc-writer | qa-script-writer | TC file ready - start data + spec |
| `MODULE-DONE: qa-tc-writer \| {module}` | qa-tc-writer | QA Lead | Module cases written + verified |
| `SECTION-DONE: {module} \| case_ids: ...` | qa-tc-writer | qa-hawk, QA Lead | Imported - hawk reads case_ids for smoke |
| `PROGRESS: qa-tc-writer \| {module} \| N/total` | qa-tc-writer | QA Lead | Import progress (every 5 cases) |
| `META: testrail_run_id={id}` | qa-tc-writer | qa-hawk | Run created - flush pending TR results |
| `HAWK: environment \| READY` | qa-hawk | QA Lead | Site healthy - unblocks all agents |
| `HAWK: smoke \| {module} \| PASS/FAIL` | qa-hawk | QA Lead | Smoke gate result |
| `HAWK: smoke-result-pending \| {module}` | qa-hawk | qa-hawk | Held - waiting for run_id |
| `HAWK: tr-result-pending \| {module}` | qa-hawk | qa-hawk | Bug result held - waiting for run_id |
| `HAWK: explore \| {module} \| ...` | qa-hawk | QA Lead | Explore complete + confidence level |
| `FLAKY: qa-hawk \| {module} \| ...` | qa-hawk | QA Lead | Intermittent failure found |
| `FLAKY: qa-script-writer \| {module} \| ...` | qa-script-writer | QA Lead | Flaky spec detected (3-run check) |
| `PROGRESS: qa-script-writer \| Phase 0 bootstrap complete` | qa-script-writer | QA Lead | Bootstrap done |
| `PROGRESS: qa-script-writer \| CI config generated` | qa-script-writer | QA Lead | GitHub Actions yaml written |
| `BLOCKED: {agent} \| {reason}` | any agent | QA Lead | Hard blocker - only QA Lead surfaces to user |
| `DONE: qa-tc-writer` | qa-tc-writer | QA Lead | All modules imported, run created |
| `DONE: qa-script-writer \| ... \| matrix: ...` | qa-script-writer | QA Lead | Specs running, artifacts ready |
| `DONE: qa-hawk` | qa-hawk | QA Lead | Testing complete, bugs filed |
| `RESULT: {JIRA-KEY} \| PASS/FAIL` | qa-hawk | QA Lead | Retest result |

---

## Integrations

| Integration | Used for | Required |
|-------------|----------|----------|
| **Atlassian (Confluence)** | Fetch sprint docs, BRD, release notes; host bug report page (no-Jira mode) | Yes (MCP plugin) |
| **Atlassian (Jira)** | Sprint health scan, story status tracking, bug creation, retest triggers | Optional |
| **TestRail** | Sections, test cases, test run, per-case results | Optional (skipped if no URL) |
| **Postman** | Collection creation, API test import, workspace management | Optional |
| **Newman** | Run Postman collection via CLI | Optional (auto-installed by qa-api-tester) |
| **GitHub Actions** | CI config generated automatically (Playwright suite) | Generated - no setup needed |

---

## Prerequisites

- **Claude Code** - CLI or desktop app
- **MCP Plugins installed and enabled:**
  - `Atlassian` - for Confluence + Jira access
  - `Playwright` - for browser automation during qa-hawk exploration
  - `Postman` - for API collection management
- **Node.js 18+** - for Playwright automation (qa-script-writer installs everything else)
- A staging/QA environment URL accessible from the machine running Claude Code

---

## Setup

**1. Clone / copy this project into your working directory**

**2. Copy `.env.example` to `.env` and fill in your values:**

```bash
cp .env.example .env
```

Minimum required for a first run:
```
STAGING_URL=https://staging.yourapp.com
TEST_ADMIN_EMAIL=admin@test.com
TEST_ADMIN_PASSWORD=yourpassword
```

Everything else (TestRail, Jira, Postman) is optional. QA Lead will ask for any missing credentials at the start of each sprint.

**3. Open Claude Code in this directory and run:**

```
/qa-lead {confluence-url-or-paste-release-note}
```

That's it. QA Lead handles the rest.

---

## Configuration Reference

| Variable | Purpose | Required |
|----------|---------|----------|
| `STAGING_URL` | Base URL of the environment under test | Yes |
| `API_BASE_URL` | Base URL for API calls in specs (e.g. `https://api.yourapp.com`) | Yes |
| `TEST_ADMIN_EMAIL` | Admin test account | Yes |
| `TEST_ADMIN_PASSWORD` | Admin test account password | Yes |
| `TEST_USER_EMAIL` | Standard user test account | Optional |
| `TEST_USER_PASSWORD` | Standard user test account password | Optional |
| `JIRA_PROJECT_KEY` | Jira project for bug creation and story tracking | Optional |
| `JIRA_READY_FOR_QA_STATUS` | Status name that triggers qa-hawk smoke | Optional |
| `CONFLUENCE_SPACE_KEY` | Default space for doc search and bug pages | Optional |
| `TESTRAIL_BASE_URL` | TestRail instance URL | Optional |
| `TESTRAIL_USERNAME` | TestRail login email | Optional |
| `TESTRAIL_API_KEY` | TestRail API key | Optional |
| `TESTRAIL_PROJECT_ID` | Target project ID | Optional |
| `TESTRAIL_SUITE_ID` | Target suite ID | Optional |
| `POSTMAN_API_KEY` | Postman API key for collection management | Optional |
| `POSTMAN_WORKSPACE_ID` | Target Postman workspace | Optional |

---

## Agent Autonomy Rules

After the user types `proceed` on the test plan, the pipeline runs with **full autonomy** until the sign-off report. The four moments QA Lead will pause and speak to the user:

1. **Phase 0** - a required credential is missing from `.env`
2. **Phase 0** - Playwright MCP browser tools are not permitted (preflight check fails)
3. **Hard blocker** - staging unreachable >30 min, or Atlassian auth failing
4. **Sign-off** - final report presented for approval

Every other decision (gap filling, import retries, TestRail section creation, bug classification, retest triggering) is handled autonomously and silently.

---

## Sign-off Report

The final output includes:

```
DECISION GATE RECORD
Open Critical bugs        → must be Zero
Test pass rate            → must be ≥95%
E2E automation coverage   → must be ≥80%
Flaky tests               → must be ≤3

AUTOMATION QUALITY
Flaky tests list
Performance flags (pages > 3s load)
Accessibility violations (WCAG 2.1 AA)
CI config status

RECOMMENDATION: PASS / CONDITIONAL / FAIL
```

---

## Project Structure

```
AG-QREW/
├── .claude/
│   ├── commands/
│   │   ├── qa-lead.md          Orchestrator skill
│   │   ├── qa-tc-writer.md     Test case writer skill
│   │   ├── qa-script-writer.md Playwright automation skill
│   │   ├── qa-hawk.md          Exploratory tester skill
│   │   └── qa-api-tester.md    API tester skill
│   └── settings.json           Claude Code permissions
├── .env.example                Environment variable template
├── .gitignore                  Excludes .env and qa/ runtime output
├── CLAUDE.md                   Pipeline rules (agent spawning, signals, gates)
└── README.md                   This file
```

> `qa/` is generated fresh each sprint and excluded from git.

---

## Key Design Decisions

**Why a shared text file for coordination?**
`qa/shared-task-list.txt` is the single source of truth. Every agent reads and writes it. It's human-readable, durable across agent restarts, and makes the entire pipeline state inspectable without a database or message broker.

**Why sequential TC import (not parallel)?**
TestRail rate-limits aggressive parallel POST requests. Sequential import with 500ms pacing and 3-retry logic proved more reliable than batching, especially for large modules (50+ cases).

**Why `@mobile` tags instead of separate spec files?**
Mobile TC blocks live in the same `.txt` file as other TCs for a module. Tagging them `@mobile` in the spec name lets Playwright's `grep` route them to the correct viewport projects (`playwright.config.ts`) without duplicating any test logic.

**Why axe/perf checks in `navigate()` as soft warnings?**
Accessibility and performance are non-blocking concerns that shouldn't fail a functional test. Logging them via `console.warn` surfaces them in the playwright-report and sign-off report without polluting the pass/fail signal.

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---------|-------------|-----|
| Playwright MCP tools blocked mid-run | Browser permissions not in `settings.json` | Add Playwright tools to `.claude/settings.json` allowedTools - QA Lead now checks this in Phase 0 before spawning any agent |
| `BLOCKED: qa-script-writer \| browser-snapshot-failed` | Browser access denied or returned empty accessibility tree | Same as above - grant permissions and re-run qa-script-writer Phase 1A |
| `BLOCKED: qa-hawk \| auth-failed` | `.env` credentials wrong or staging account locked | Fix `TEST_ADMIN_EMAIL` / password in `.env` |
| `IMPORT-FAILED: TC-NNN after 3 retries` | TestRail API key expired or rate limit | Check TestRail API key; pipeline continues, failed cases logged |
| `Cannot find module '@playwright/test'` | Phase 0 npm install didn't run or failed | Re-trigger qa-script-writer; it will re-run Phase 0 (idempotent) |
| TestRail cases all in one flat section | Old pipeline run before Issue 2 fix | Cases from new runs will be nested; re-import to fix old runs |
| `FLAKY: qa-script-writer` signals appearing | Selector instability or staging latency | Add `test.slow()` or fix the selector; check AUTOMATION QUALITY in sign-off |
| QA Lead asking user mid-pipeline | A BLOCKED signal from an agent | Read the signal, resolve the blocker, signal QA Lead to continue |
