# QA Pipeline Rules

## Agent Spawning — ALWAYS use Agent tool, NEVER Skill tool for sub-agents

qa-tc-writer, qa-script-writer, qa-api-tester, qa-hawk are SKILLS — not
built-in subagent types. The Skill tool is sequential and blocks the pipeline.

For true parallelism, spawn each as a general-purpose Agent with
run_in_background=true. In the prompt, tell the agent to read its own
skill file first — background agents have full filesystem access.
Send all in ONE message so they fire simultaneously:

```
`Agent`(subagent_type="general-purpose", `run_in_background=true`,
      description="qa-tc-writer: write test cases",
      prompt="Read your skill instructions from .claude/commands/qa-tc-writer.md
              then execute them. Task context: [modules, TestRail config, shared-task-list path]")

`Agent`(subagent_type="general-purpose", `run_in_background=true`,
      description="qa-script-writer: write E2E scripts",
      prompt="Read your skill instructions from .claude/commands/qa-script-writer.md
              then execute them. Task context: [site URL, module list, watch for TC-READY signals]")

`Agent`(subagent_type="general-purpose", `run_in_background=true`,
      description="qa-api-tester: test API endpoints",
      prompt="Read your skill instructions from .claude/commands/qa-api-tester.md
              then execute them. Task context: [API docs URL, Postman workspace]")
```

Background agents do NOT auto-load skills — they must be told to read their
skill file via the prompt. The Skill tool is only for qa-lead itself.

## Agent Spawn Gate — NEVER spawn before test plan is approved

Agents are spawned ONLY after the test plan is complete and the user has
approved it. The sequence is strictly:

```
Phase 0 → collect config & credentials  (ask user if missing)
Phase 0 → browser preflight check       (see below — must pass before Phase 1)
Phase 1 → generate test plan            (present to user, wait for "proceed")
Phase 2 → spawn all agents              (ONE message, all fire simultaneously)
```

Do NOT spawn any agent during Phase 0 or Phase 1.
Do NOT spawn agents one by one — all go in a single message at Phase 2 start.
If the user has not typed "proceed" on the test plan, the gate is closed.


## Phase 0 Preflight — Browser Access Check (qa-lead only)

Before spawning any agent, qa-lead must verify Playwright MCP browser tools
are permitted. Do a single test call:

```
browser_navigate(url="{staging_url}")
browser_snapshot()
```

If either call fails or returns a permission error → surface to user immediately:
  "Playwright MCP browser tools are not permitted. Add them to .claude/settings.json
   under allowedTools before proceeding."
Do NOT spawn any agent until browser access is confirmed working.

This prevents agents from discovering the blockage mid-run and silently
degrading to guessed locators.


## Sub-Agent Silence Rules — agents NEVER prompt the user

Every background agent (qa-tc-writer, qa-script-writer, qa-api-tester, qa-hawk)
must operate fully silently. They communicate ONLY via shared-task-list.txt.

| Situation | What the agent does |
|---|---|
| Ambiguous requirement | Use most conservative testable interpretation, proceed |
| Missing test data | Generate minimal valid data, proceed |
| Page not loading | Retry 3×, then write BLOCKED: {agent} {reason} to shared-task-list.txt |
| Auth failure | Write BLOCKED: {agent} auth-failed to shared-task-list.txt |
| Any other blocker | Write BLOCKED: {agent} {reason}, stop and wait |

QA Lead monitors shared-task-list.txt and is the ONLY one who may surface a
blocker to the user. Sub-agents never talk to the user directly.


Agents coordinate through `qa/shared-task-list.txt` signals:
- TC-READY: {module}                      → qa-script-writer picks up that module immediately
- MODULE-DONE: {module}                   → qa-tc-writer imports to TestRail immediately (autonomous)
- SECTION-DONE: {module} | case_ids: ...  → QA Lead verifies TestRail count silently; qa-hawk reads case_ids for smoke recording
- PROGRESS: qa-tc-writer | {module} | N/total → import rate-limit monitor (every 5 cases)
- HAWK: smoke-result-pending | {module}   → held result waiting for testrail_run_id; posted when run_id appears
- HAWK: tr-result-pending | {module}      → held bug→TestRail result; posted when run_id appears
- META: testrail_run_id={id}              → written by qa-tc-writer after creating the run; qa-hawk flushes all pending results
- DONE: qa-tc-writer                      → all modules imported, test run created
- FLAKY: qa-script-writer | {module} | … → flakiness detection result (Phase 3 Step 5); feeds sign-off gate
- DONE: qa-script-writer                  → specs written + run; coverage-matrix.txt + CI config ready
- DONE: qa-hawk                           → QA Lead runs bug consolidation silently

### What happens when qa-tc-writer signals DONE

This is fully automated. No user input at any step:

```
1. qa-tc-writer creates the test run immediately after all modules are imported
2. qa-tc-writer writes META: testrail_run_id={run_id} to shared-task-list.txt
3. qa-tc-writer writes DONE: qa-tc-writer signal
4. qa-hawk (already running per-module smoke + explore) reads testrail_run_id
   and flushes any HAWK: smoke-result-pending and HAWK: tr-result-pending records
   that were held because run_id was not yet available
5. Pipeline continues — user is NOT notified
```

Note: qa-hawk does NOT wait for DONE: qa-tc-writer before updating TestRail.
It updates results in real-time per TC as it executes them (Mode 2 Step 1) and
per bug as it logs them (Mode 2 Step 3b). The run_id flush in step 4 above only
covers results that were held because the run hadn't been created yet.

QA Lead must NOT ask:
- "Should I create the test run?"
- "Want to review the cases before I proceed?"
- "tc-writer is done — what next?"

All three are auto-resolved. The test run is always created by qa-tc-writer immediately on completion.


## TC Verification Loop — built into qa-tc-writer skill, fully silent

The 2-round self-verification is Step 3 of qa-tc-writer's own skill file
(.claude/commands/qa-tc-writer.md). It runs automatically after every
module. QA Lead does NOT challenge qa-tc-writer externally — that created
the sequential bottleneck. qa-tc-writer challenges itself.

```
write cases → Round 1 gap check → fill gaps → Round 2 confirm → import → MODULE-DONE → next module
```

Round 1: re-reads TC file, identifies every uncovered field/button/error/
         boundary/role, adds missing cases directly — no reporting, no asking.
Round 2: re-reads updated file, confirms gaps are filled, adds any remainder.
         When satisfied → posts MODULE-DONE + TC-READY, imports, moves on.

This loop is OWNED by qa-tc-writer, not QA Lead.
QA Lead does NOT review cases between modules.

qa-tc-writer must NOT ask:
- "I found N gaps — should I add them?"
- "Round 1 complete — proceed to Round 2?"
- "Coverage looks good — shall I import?"


## Internal Loops — NEVER surface to user

The following are fully internal and silent — the user must never be prompted:

| Internal process | How to handle |
|---|---|
| TC verification Round 1 & 2 | Owned by qa-tc-writer — silent, self-resolved |
| TestRail import pacing | qa-tc-writer: 500ms sleep + 3 retries per case; PROGRESS every 5 |
| Smoke pass/fail per module | qa-hawk posts result to every case_id in SECTION-DONE immediately |
| TestRail result per TC execution | qa-hawk posts pass/fail per case as it runs Mode 2 Step 1 |
| Bug found → TestRail update | qa-hawk posts Failed result immediately (Step 3b) — never batched |
| Pending result flush | qa-hawk flushes held results when META: testrail_run_id appears |
| Import verification (TestRail count) | curl verify, log, continue |
| Agent DONE signal processing | Read, validate format, push bugs silently |
| Gap analysis assumptions | Document in test plan Section 8, proceed |
| Flakiness detection | qa-script-writer Phase 3 Step 5 — silent; writes FLAKY: signals only |
| A11Y + perf scan on navigate() | qa-script-writer — console.warn only; never fails a test |
| Coverage matrix write | qa-script-writer Phase 3 Step 7 — silent; QA Lead reads at sign-off |


## The ONLY times to prompt the user

1. Missing credentials or config during Phase 0 setup
2. Playwright MCP browser access fails in preflight check
3. Hard blockers: staging completely unreachable >30 min, Atlassian auth failing
4. Final sign-off: present the sign-off report and ask for approval

After the user types "proceed" past any gap questions — FULL AUTONOMY until sign-off.


## Project Config

Fill this in during Phase 0 setup. QA Lead reads it at sprint start.

| Setting | Value |
|---|---|
| Site URL | |
| API Base URL (`API_BASE_URL` in .env) | |
| Swagger / API docs URL | |
| Confluence space | |
| TestRail base URL | |
| TestRail username | |
| TestRail API key | |
| TestRail project_id | |
| TestRail suite_id | |
| TestRail milestone_id | |
| TestRail custom_case_environment | (omit if not used by your instance) |
| Postman workspace | |
| Jira project | (leave blank if bugs go to Confluence) |
| Auth required (UI) | |
