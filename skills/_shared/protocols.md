# Shared Protocols

## Skill Layout Contract

Monaiq skills should follow the same readable command shape as the GSD skills while keeping Monaiq's route, journal, checkpoint, and evidence requirements explicit.

Monaiq guidance is distributed to Claude, GitHub Copilot / VS Code, and Codex runtimes. Shared guidance must therefore describe behavior in target-host terms instead of assuming one runtime's tool names apply everywhere.

Use this section order for user-facing skills unless a read-only reference skill has a good reason to be shorter:

1. Frontmatter with `name`, `description`, `agent: monaiq`, `auto-invoke` when useful, `tags`, `category`, `allowed-tools`, `tier`, and `invoked-by`.
2. `<objective>` with the user outcome and mutation boundary.
3. `<input-context>` and `<output-context>` for route packets, evidence packets, and downstream handoffs.
4. `<execution_context>` naming the shared workflow files and the skill's narrow responsibility.
5. `<monaiq-agent-handoff>` using the standard degraded-orchestration warning when direct invocation cannot activate the custom agent.
6. Optional authority blocks, such as `<tool-first-authority>`, only when a hosted MCP tool or canonical platform resource is authoritative.
7. `<workflow>` as numbered executable steps that start with startup/journal readiness and end with completion/handoff.
8. `<reference>` for domain facts, decision tables, examples, and platform resource URIs.
9. `<success_criteria>` for observable completion evidence.

The required shared route packet fields are exactly `scenario`, `targetApp`, `platform`, `activePlatform`, `targetProject`, `outOfScopePlatforms`, `profileState`, `catalogState`, `codeEvidenceSummary`, `journalEvidenceSummary`, `assumptions`, `recommendedSkill`, `recommendationRationale`, `checkpointName`, and `authorizedBy`. Do not rename them in handoff text.

`platform` may describe the broad stack family (`dotnet`, `react`, `node`), while `activePlatform` is the concrete implementation target currently approved for changes (`dotnet/blazor-server`, `react/vite`, etc.). `targetProject` names the project or package that may be changed. `outOfScopePlatforms` records stacks that must not be installed or modified without a new checkpoint.

## Direct Invocation Contract

This is the canonical fallback every skill applies when invoked directly without the `monaiq` custom agent. Skills should reference this section by name rather than restating it.

**Mutation-capable skills** (catalog, SDK, feature, purchase, journal, troubleshoot fixes):
1. If the host can activate or switch to `monaiq`, hand off the current request and loaded journal state before continuing.
2. If host activation is unavailable, warn exactly once: *"Monaiq orchestration is degraded in this runtime; I will continue with the compatibility fallback, but journal startup and hard checkpoints still apply."*
3. The compatibility fallback still runs `_shared/workflows/startup.md`, uses `monaiq_journal` for `skill_started`, hard checkpoints, milestone progress, and `skill_completed`, and batches routine updates through `_shared/workflows/completion.md`.

**Read-only reference skills** (`domain-reference`, and read-only paths in `scenario-advisor` / `profile-onboarding` view modes) may answer without journal startup. They must enter the full mutation contract above the moment the user pivots to catalog, credential/config, code, or purchase work.

Business-readable output comes first. Compact technical backing follows with route packet fields, evidence summaries, checkpoint names, and no-secret implementation details. See `_shared/response-patterns.md` "Evidence Backing" for the canonical evidence shape used by specialist workflow steps.

Route, handoff, journal, and checkpoint payloads must stay secret-free. Record `session established`, credential source status, redacted placeholders, and package/version facts; never record raw ApiKeys, EncodedCredential values, JWTs, Stripe keys, `.env` contents, user-secrets contents, or encoded credential fragments.

## Response Pattern

Monaiq is a goal-driven licensing assistant. Every reply moves the user one step closer to the next journey gate. Skills reference this section by name; do not restate it in skill bodies.

**The five elements (in order, every user-facing reply).**

1. **Phase tag** — one bracketed line at the top: `[Phase: <current> → <next>]` (use `→` to signal forward motion, drop the arrow when staying in-phase, suffix `✓` on completion, suffix `· ⚠ validation` on a blocked validation).
2. **Gate line** — one sentence restating the journey gate using terms from `monaiq://protocols/implementation-journal`. Examples: *"You're at the catalog gate. Next forward action: confirm a starter catalog."* / *"Last gate cleared: onboarding ✓. Next: catalog."* / *"Validation blocked at base-sdk: provider registration missing."*
3. **Recommendation first** — state the proposed action as a concrete default, not as a question. One line. Lead with the answer; do not lead with the analysis.
4. **Evidence cite** — 1–3 short lines, each citing a concrete artifact (file, endpoint, capability, route-packet field, journal decision, catalog fact). No "based on my analysis" framing; cites only.
5. **Host-native question** — exactly one blocking ask widget with 2–4 options; one option marked recommended; one option always lets the user inspect evidence (*"Pause — show me &lt;source&gt;"*) so they can drill in without restarting.

**Recommendation-First, evidence-cited.** The order of elements 3 and 4 is binding. Recommendation comes BEFORE evidence. Evidence is a cite, not a preamble. This is the core difference between a directive assistant and a research assistant.

**Sample turn (greenfield, getting-started skill).**

```
[Phase: Onboarding → Catalog]
You're at the catalog gate. Next forward action: confirm a starter catalog so we
can scaffold the SDK.

I'd start with product "Acme Studio", two features (`manage-users` access gate,
`bulk-export` rate-limit 100/day), and three offerings (Free, Pro $19/mo,
Enterprise unlimited). This is the smallest catalog that unblocks SDK setup;
we refine pricing in the catalog gate.

Cites:
  • Project name from `Acme.Studio.csproj`
  • `[Authorize]` on `AdminController` → manage-users gate
  • `BulkExportController` 100-row default → bulk-export rate-limit

[host-native question]
  ► Approve starter catalog and proceed (recommended)
  ▸ Adjust features before approving
  ▸ Adjust pricing before approving
  ▸ Pause — show me analyze-codebase output first
```

The same five elements adapt naturally to other situations (resume, mid-skill, handoff, validation-failure, completion). The agent does not need separate templates per situation; element 2 (gate line) carries the situational variance.

**Anti-patterns (do not emit).**

- Open question as the lead: *"What features does your app have?"*, *"What licensing model do you want?"*, *"How would you like to charge?"*
- Book-report framing: *"Based on my analysis, …"*, *"Evidence suggests …"*, *"After researching …"* — strip these phrases entirely; the cite block carries the evidence
- Flat menu without recommendation: *"Here are 3 options, which would you like?"*
- Non-blocking ask: *"Let me know if you want to proceed"*, *"Feel free to pick"* — must use the host's blocking widget
- Rationale paragraph longer than one sentence; cites longer than one line each
- Reply that does not open with `[Phase: …]`
- Reply that introduces a new question without first restating the gate

**Cross-host question parity.**

| Host | Mechanism |
|---|---|
| GitHub Copilot (VS Code) | `vscode_askQuestions` — recommended flag, multi-select, free text supported |
| Claude Code | `AskUserQuestion` — equivalent option/free-text shape |
| Codex CLI / headless runtimes | structured numbered options in chat; require numeric reply before proceeding |

Skills cite "host-native question" by name and never inline a host-specific call.

**Evidence sources, in priority order.** Skills consult these before composing the recommendation:

1. Upstream route-packet inputs from the previous skill (`selectedScenario`, `capabilities`, `appType`, `quickStartSpec`, `catalogSpec`, etc.)
2. Codebase evidence — invoke or reuse `analyze-codebase` when a codebase is reachable
3. Journal evidence — prior `monaiq_journal` decisions, checkpoints, validation outcomes
4. Catalog/profile state — `product`, `product_feature`, `offering`, `feature_offering`, `profile` reads
5. Conversation cues already captured in the route packet
6. Resource patterns — `monaiq://patterns/*`, `monaiq://domain/*`, `monaiq://platforms/*`

If evidence is thin, the recommendation IS *"scope down — let me run `analyze-codebase` first"* with that as the marked option in the host-native question. Never lead with an open question because evidence is unavailable; lead with the smallest action that produces evidence.

**When the user defers ("you decide", "figure it out").** Treat as authorization to apply the recommended option. Continue under the recommendation, journal the deferral, surface the final choice at the owning checkpoint for confirmation. Do not bounce the question back — that is the failure mode this section exists to prevent.

**Platform-agnostic source.** Skills must not embed framework-specific code, package names, or API surface (e.g., React `featureId` vs .NET `FeatureKey`). Per-stack guidance is owned by `monaiq://sdk/{stack}/setup` resources and the implement-* tools' server-side responses. SDK changes update those; plugin source stays stable.

## File Operation Application Protocol

When a `monaiq_journal` call returns a `fileOperations` array, the calling agent MUST apply every operation before continuing. This is the single authoritative definition; skills reference it by name instead of restating it.

1. Receive the `fileOperations` array from the `monaiq_journal` tool response.
2. For each operation in the array, in order:
   - If `type == "ensureDirectory"`: ensure the directory at `path` exists — create it if it does not exist.
   - If `type == "writeFile"`: create or overwrite the file at `path` with the value of `content`.
   - If `type == "appendFile"`: append the value of `content` to the file at `path`, creating it if absent.
3. After applying all operations, verify each `writeFile` path exists on disk.
4. If any write cannot be confirmed, report the failure immediately and do not continue the skill workflow.

Validate every path against the `.monaiq` allowlist before applying: `.monaiq/`, `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, `.monaiq/config.json`, `.monaiq/CHECKPOINTS`, and `.monaiq/CHECKPOINTS/*.md`. Reject operations that target paths outside this allowlist.

## Host-Native Ask Pattern

When a skill step requires a blocking user decision, use the target host's native ask mechanism. This section is the authoritative call-syntax reference; `checkpoint.md` and skill workflow steps reference it by name. Skills use `host-native question` as the canonical term in workflow prose; host-specific call syntax appears only here and in `checkpoint.md`.

**VS Code / GitHub Copilot** — call `vscode_askQuestions`:

```json
vscode_askQuestions([{
  "header": "Monaiq — [GATE-NAME]",
  "question": "[Question text shown to the user]",
  "options": [
    { "label": "[Option A — recommended]", "value": "option-a", "recommended": true },
    { "label": "[Option B]", "value": "option-b" },
    { "label": "Pause — show me [source]", "value": "pause" }
  ]
}])
```

**Claude / Claude Code** — call `AskUserQuestion` with the same shape:

```json
AskUserQuestion([{
  "header": "Monaiq — [GATE-NAME]",
  "question": "[Question text shown to the user]",
  "options": [
    { "label": "[Option A — recommended]", "value": "option-a", "recommended": true },
    { "label": "[Option B]", "value": "option-b" },
    { "label": "Pause — show me [source]", "value": "pause" }
  ]
}])
```

**Codex / headless runtimes** — present a numbered list in chat; require a numeric reply before proceeding:

```
1. [Option A — recommended]
2. [Option B]
3. Pause — show me [source]

Reply with a number (e.g., "1") to continue.
```

Do not continue the skill workflow until the user's answer is received. Do not infer approval from conversational text.

## Tool Operation Rules

- `design-monetization`: read/list catalog facts only. Do not create, update, publish, or delete products, features, offerings, or feature assignments.
- `scenario-advisor`: read/list only plus resource reads.
- `analyze-codebase`: local read/search plus resource reads; no catalog mutation.
- `domain-reference`: resource reads only.
- `profile-onboarding`: profile/session reads and profile-directed onboarding only; no catalog or code mutation.
- `manage-catalog`: product, product_feature, offering, and feature_offering create/update/delete are allowed only after `CHECKPOINT-PRE-CATALOG-MUTATION` or publication-specific checkpoint approval.
- `implement-licensing`: code/config edits are allowed only after required SDK resources, checkpoint approval, and secret-safety checks.
- `implement-feature`: business-logic edits are allowed only after `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT` and tool/resource evidence.
- `implement-purchase-flow`: checkout edits and credential persistence are allowed only after SDK/tool/resource evidence and purchase-specific checkpoints.
- `troubleshoot-integration`: read/diagnose first; fixes that mutate code/config/business logic require the validation and checkpoint workflows.

When a skill has broad tool exposure for platform compatibility, these operation rules are the binding behavioral constraint.