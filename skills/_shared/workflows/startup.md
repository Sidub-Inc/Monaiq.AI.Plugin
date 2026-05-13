# Startup Workflow

Use this workflow before catalog mutation, SDK/config edits, feature implementation, purchase-flow work, troubleshooting fixes, or validation remediation.

1. Resolve readiness for `fetch_step_resources`, `monaiq_journal`, and any required MCP implementation tool for the selected skill.
2. Fetch `monaiq://protocols/implementation-journal` and read `ROUTING-MAP.md`, `_shared/handoff-schemas.md`, and `_shared/response-patterns.md` when available.
3. Reuse a current journal state packet supplied by the previous Monaiq skill in this turn when it includes current skill, next action, checklist state, and recent activity. Otherwise call `monaiq_journal get_state` for the consumer project root.
4. If state is missing or stale, call `monaiq_journal init` and apply returned file operations using the **File Operation Application Protocol** in `_shared/protocols.md`:
   - For each `ensureDirectory` operation: ensure the directory at `path` exists.
   - For each `writeFile` operation: create or overwrite the file at `path` with `content`.
   - After applying, verify that `.monaiq/STATE.md` and `.monaiq/JOURNAL.md` exist and `.monaiq/CHECKPOINTS` directory exists.
   - If any expected file is absent after application, report the failure and do not continue.
   > **Scope note:** The **File Operation Application Protocol** applies to every `monaiq_journal` call throughout skill execution — not only the `init` call at startup. Whenever a journal call returns `fileOperations`, apply them immediately before continuing the workflow.
5. Verify `.monaiq/STATE.md` contains the master journey checklist. Restore it through the journal protocol before substantive work if absent.
6. Call `monaiq_journal skill_started` only when starting a new user-visible journey, resuming after stale or missing state, or switching to a materially different specialist without a handoff packet. Do not emit lifecycle journal calls for every internal handoff when the caller already passed fresh state.
7. For new or materially changed journeys, save and present `CHECKPOINT-WORKFLOW-START` before substantive work.
8. Route to the narrowest missing prerequisite when profile/session, catalog/offering, SDK, code evidence, journal state, or MCP readiness is missing.

After workflow start and after every catalog mutation, SDK setup, secondary-platform, or validation-failure checkpoint, persist a no-secret route/state packet containing `activePlatform`, `targetProject`, and `outOfScopePlatforms` in `.monaiq/STATE.md` through `monaiq_journal`. Downstream skills must preserve these fields unless a new host-native checkpoint changes them.

## Journal Cadence

Journal updates are milestone records, not a transcript of every tool call. Prefer one startup state read, hard checkpoint result writes, validation-failure writes, and one completion/progress batch per meaningful skill outcome. Combine routine route, decision, changed-path, checklist, and handoff summaries when they describe the same milestone.

Do not call `monaiq_journal` between every other tool call, solely to mirror a tool response, or solely because `journalReadyUpdates` exists. Treat `journalReadyUpdates` as a queue of journal intents to coalesce into the next milestone update unless it contains a blocker, validation failure, or hard checkpoint requirement.

Direct skill invocation is first-class. The custom `monaiq` agent is an optional orchestrator, not a correctness dependency.

## Readiness Blocker Report

When required MCP capabilities are unavailable, render a user-visible blocker report instead of continuing into half-execution:

- **Missing:** unavailable tool, resource, or runtime capability.
- **Affected gate:** checklist gate or skill blocked.
- **What can continue:** read-only analysis, domain explanation, or recommendation-only work.
- **What is blocked:** catalog mutation, credential/config writes, application behavior edits, checkpoint recording, or validation remediation.
- **Recovery:** refresh/reload the Monaiq plugin or rerun with a working MCP server.
- **Journal:** record a no-secret diagnostic summary when `monaiq_journal` is available.