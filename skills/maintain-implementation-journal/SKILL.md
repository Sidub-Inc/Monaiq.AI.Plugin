---
name: maintain-implementation-journal
description: "Use when: maintaining Monaiq implementation state, checkpoints, route packets, journal evidence, checklist progress, validation failures, or resumable handoffs."
agent: monaiq
tags: [journal, checkpoints, state, handoff]
category: orchestration
tier: 2
invoked-by:
  - getting-started
  - troubleshoot-integration
  - analyze-codebase
  - design-monetization
  - manage-catalog
  - profile-onboarding
  - scenario-advisor
  - implement-licensing
  - implement-feature
  - implement-purchase-flow
allowed-tools:
  - monaiq_journal
  - fetch_step_resources
  - mcp__plugin_monaiq_monaiq__monaiq_journal
  - mcp__plugin_monaiq_monaiq__fetch_step_resources
---

# Maintain Implementation Journal

<objective>
Maintain the durable `.monaiq/` implementation journal that lets Monaiq skills collaborate across sessions without replaying broad intake or losing approval evidence. This skill owns journal layout, checkpoint anatomy, file-operation application, credential safety, resume packets, route packet persistence, checklist progress, and validation-failure recording.
</objective>

<execution_context>
This is the authoritative protocol resource for `monaiq://protocols/implementation-journal`. Every consequential Monaiq skill should read this file with `ROUTING-MAP.md`, `_shared/protocols.md`, `_shared/workflows/startup.md`, `_shared/workflows/checkpoint.md`, `_shared/workflows/completion.md`, and `_shared/workflows/validation.md` before catalog, credential/config, code, purchase, or remediation work.
</execution_context>

<input-context>
Receives safe route context, current skill name, checkpoint prompts/results, file-change summaries, validation outcomes, backend read-back facts, and downstream handoff packets from Monaiq skills and MCP tool envelopes.
</input-context>

<output-context>
Provides `.monaiq/STATE.md` as the compact resume/control packet, `.monaiq/JOURNAL.md` as the append-only evidence ledger, `.monaiq/CHECKPOINTS/*.md` as immutable approval/result evidence, and optional non-secret `.monaiq/config.json` hints for future sessions.
</output-context>

<workflow>
1. Start from a current journal state packet when the previous Monaiq skill already loaded one in the same turn; otherwise call `monaiq_journal get_state`.
2. If state is absent or stale, call `monaiq_journal init` and apply returned file operations using the **File Operation Application Protocol** in `_shared/protocols.md`. Verify that `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, and `.monaiq/CHECKPOINTS` all exist after application; if any are missing, report the failure before continuing.
3. Call `skill_started` only for a new user-visible journey, stale or missing state recovery, or a materially different specialist handoff that lacks a current state packet.
4. Use `save_checkpoint` for every `CHECKPOINT-*` prompt/result pair, then apply returned file operations using the **File Operation Application Protocol** in `_shared/protocols.md`. Verify the expected `.monaiq/CHECKPOINTS/*.md` file exists before claiming the checkpoint is recorded.
5. Use `update_checklist_progress`, `record_file_changes`, `record_validation_failure`, and journal append actions only with evidence-backed, no-secret summaries. Coalesce routine route, decision, changed-path, checklist, and handoff updates into milestone-sized writes.
6. Before handoff or exit, save meaningful completion proof only when the completion is consequential, apply returned operations, verify expected files, then call `skill_completed` once for the user-visible skill outcome.
</workflow>

<journal-contract>
Maintain a `.monaiq/` folder at the consumer project root while Monaiq skills guide implementation. The canonical layout is exactly `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, `.monaiq/CHECKPOINTS/*.md`, and optional non-secret `.monaiq/config.json`. This is the optional non-secret .monaiq/config.json slot for local, non-credential configuration hints only.

Read `ROUTING-MAP.md` and `maintain-implementation-journal.md` as the first shared contract sources before a skill invents local route, checkpoint, or resume vocabulary. `ROUTING-MAP.md` owns shared journey vocabulary and route packet phase boundaries; `maintain-implementation-journal.md` owns checkpoint anatomy, approval/result recording, and journal state transitions.

Use `monaiq_journal` as the canonical write path. It returns `fileOperations` for the calling agent to apply locally under the consumer project root. Direct agent writes are not a replacement fallback when the hosted tool is unavailable or stale.

Record skill lifecycle, consequential decisions, todos, blockers, questions, validation failures, checkpoint prompts/results, errors, and code-change summaries. Decisions, todos, blockers, questions, validation failures, and code-change summaries use durable append-only IDs in `JOURNAL.md`. File-change summaries contain paths and intent only, not file contents. Routine journal writes should be sparse milestone records rather than a tool-call transcript.

Tool envelopes may include `journalReadyUpdates`. Treat them as proposed journal intents, not applied state. Queue and coalesce supported intents into the next milestone journal update unless an intent records a blocker, validation failure, or hard checkpoint requirement that must be durable before proceeding. Apply the returned file operations locally after path validation.
</journal-contract>

<master-journey-checklist>
The journal protocol owns the Monaiq master journey checklist. The three-layer contract is: STATE.md is the compact resume/control packet, JOURNAL.md is the append-only evidence ledger, and CHECKPOINTS/*.md is immutable approval/result evidence.

`STATE.md` includes the standard adaptive checklist template with ordered gates for onboarding/profile, catalog and offerings, base SDK setup, feature implementation, purchase flow when applicable, journal/checkpoints, validation, and completion evidence. The checklist adapts through variant branches for platform, app scenario, catalog shape, credential ownership, feature type, purchase surface, and validation depth instead of reopening avoidable tactical questions.

Use `monaiq_journal update_checklist_progress` to project checklist progress. The evidence-backed completion rule is strict: mark a gate complete only after the relevant source skill, MCP tool, canonical resources, and checkpoint or journal evidence have been used and recorded. The request must include the gate, status, and evidence summary; completion without evidence is invalid.

Routine checklist and journal updates are background operations. Do not ask the user for permission to initialize .monaiq, record routine progress, append journal entries, or mark evidence-backed checklist progress. User-facing approval remains reserved for hard checkpoints and consequential changes. Background does not mean frequent: batch routine progress so the journal captures decisions, actions, progress, blockers, and validation outcomes without slowing the workflow.
</master-journey-checklist>

<operation-application-protocol>
Calling `monaiq_journal` is incomplete until the caller applies and verifies the returned operations. You must apply returned file operations locally under the consumer project root before claiming journal progress.

For every `monaiq_journal` response:
1. Inspect `fileOperations` before continuing.
2. You must validate each fileOperations path against the `.monaiq` allowlist: `.monaiq/`, `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, `.monaiq/config.json`, `.monaiq/CHECKPOINTS`, and `.monaiq/CHECKPOINTS/*.md`.
3. Apply only supported operation kinds: `ensureDirectory`, `writeFile`, and `appendFile`.
4. Resolve paths under the consumer project root and never outside it.
5. You must verify .monaiq/STATE.md exists when state is initialized or updated.
6. You must verify .monaiq/JOURNAL.md exists when journal entries are initialized or appended.
7. You must verify .monaiq/CHECKPOINTS exists when init or checkpoint operations are returned.
8. You must verify .monaiq/CHECKPOINTS/*.md exists after a `save_checkpoint` call with a `result` payload.

Canonical checkpoint prompt call:

```json
{"checkpoint":"CHECKPOINT-PRE-CREDENTIAL-WRITE","summary":"Credential target selected.","question":"Approve the credential write plan?"}
```

Canonical approval recording call:

```json
{"checkpoint":"CHECKPOINT-PRE-CREDENTIAL-WRITE","summary":"Credential target selected.","question":"Approve the credential write plan?","result":{"answer":"approved"}}
```

Do not use checkpointName; the hosted parser reads checkpoint.
</operation-application-protocol>

<state-detection>
At Step 1 of a Monaiq journey or stale resume, call `monaiq_journal get_state`. If no journal exists, call `monaiq_journal init`, apply the returned file operations, then call `skill_started`. If a previous Monaiq skill in the same turn handed off a fresh state packet, reuse it instead of calling `get_state` again.

If state exists, read the bounded resume packet source. `STATE.md` includes current skill/step, scenario, route packet summary, last checkpoint, last activity, open todo/question/blocker/validation status, recent changed areas, and explicit next action. When `currentSkill` is set, summarize the current skill, current step, scenario, route packet summary, last checkpoint, open todos, open questions, validation status, blockers, recent changed areas, next action, and recent activity before continuing.
</state-detection>

<checkpoint-protocol>
When a step declares a `CHECKPOINT-*` checkpoint, call `monaiq_journal save_checkpoint` with the checkpoint name, summary, question, intended change, user/business impact, affected entities/files/areas, assumptions, secret policy, risks, validation plan, and options. The default sign-off choices are exactly: `approve automatic implementation`, `show a reviewable plan first`, `revise scope`, `stop`. Present the returned envelope through the host-native ask mechanism, such as `vscode_askQuestions`, Claude `ask`, or Codex `AskUserQuestion`.

Checkpoint prompts must be complete before the user is asked to approve consequential work. Present the business-readable summary, question, intended change, user/business impact, affected entities/files/areas, assumptions, secret policy, risks, validation plan, and options; keep technical identifiers available in compact backing fields rather than making them the primary user-facing explanation.

Do not treat an envelope as approval. Approval exists only after the user responds and a follow-up `save_checkpoint` call records the `result` payload. Required checkpoint examples include `CHECKPOINT-WORKFLOW-START`, `CHECKPOINT-STATE-CONFLICT`, `CHECKPOINT-FRAMEWORK-CHOICE`, `CHECKPOINT-CREDENTIAL-SOURCE`, `CHECKPOINT-PRE-BROWNFIELD-MIGRATION`, `CHECKPOINT-PRE-CATALOG-MUTATION`, `CHECKPOINT-PRE-CREDENTIAL-WRITE`, `CHECKPOINT-FEATURE-SELECTION`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, `CHECKPOINT-CHECKOUT-ARCHITECTURE`, `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE`, `CHECKPOINT-PRE-APIKEY-EXPOSURE-RISK`, and `CHECKPOINT-SKILL-COMPLETE`.

Validation failures use a failure-specific checkpoint recording failure context, likely cause, remediation options, and selected retry/change/stop path before further catalog, code, config, or remediation work.

When validation fails after code/config work, call `monaiq_journal` action `record_validation_failure` with `validationCommand`, `observedResult`, `likelyCause`, `blockedNextAction`, `retryChoices`, and `checkpointRequired`. Pause before more edits until the remediation checkpoint is resolved.
</checkpoint-protocol>

<degraded-orchestration>
If the custom agent cannot be activated, the compatibility fallback still uses this protocol. If `monaiq_journal` is missing, stale, or unavailable, skills may continue read-only guidance but must halt before catalog mutation, code edits, credential/config writes, or validation remediation rather than writing `.monaiq` directly.
</degraded-orchestration>

<resume-and-reconciliation>
Backend wins for catalog, profile, license, billing, product, offering, and credential facts. Local wins for journey progress, implementation decisions, todos, questions, checkpoints, user preferences, and handoff notes.

When backend state and local journal state diverge, update the journal to match backend-canonical facts and raise `CHECKPOINT-STATE-CONFLICT` before proceeding. Never let `.monaiq` override backend truth.

backend/catalog changes after completed implementation can reopen journey work as a coherent next action while preserving completed discovery and approval history.
</resume-and-reconciliation>

<credential-safety>
`.monaiq/` is committed by default so teams and future agent sessions can understand the implementation journey. Opt-out via `.gitignore` is possible but is not the default.

Credential safety is prose-only in this phase. Do not journal ApiKeys, EncodedCredential strings, JWT values, Stripe keys, PEM blocks, private keys, license credentials, user secrets, `.env` contents, or secret-bearing file contents. Mention paths and decisions, never raw credential values. This phase does not implement a server-side credential leak filter.

Do not place raw ApiKeys, EncodedCredential values, JWTs, Stripe keys, or secret-bearing file contents in prompts, journals, checkpoint result payloads, plugin output, or generated guidance.
</credential-safety>

<file-operation-contract>
`monaiq_journal` is deterministic C# MCP/file-operation logic. It does not write arbitrary server filesystem paths, does not add a local transport, and does not operate outside `.monaiq/`. The calling agent applies returned operations locally after validating they target `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, `.monaiq/CHECKPOINTS/*.md`, or `.monaiq/config.json`.

Do not add Semantic Kernel, Agent Framework, OpenAI, model calls, model-generated summarization, or hidden LLM orchestration inside `monaiq_journal`. The outer host LLM interprets tool responses.
</file-operation-contract>

<handoff-contract>
Before a skill exits, record changed paths only when files changed, record unresolved todos or questions, call `save_checkpoint` for `CHECKPOINT-SKILL-COMPLETE` only when meaningful, then call `skill_completed` once. If routing to another skill in the same turn, pass the fresh state and handoff packet so the next skill can avoid redundant `get_state` and `skill_started` calls unless state is stale or the route materially changes.

`CHECKPOINT-SKILL-COMPLETE` should include the no-secret `proofOfDone` packet from `_shared/workflows/completion.md` whenever files, backend state, validation status, or downstream handoff state changed. Persist routePacket, catalogSpec, pricingPlan, sdkConfig, featureImpl, checkoutImpl, and validationProof summaries according to `_shared/handoff-schemas.md` so downstream skills resume from evidence instead of re-asking.
</handoff-contract>