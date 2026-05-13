---
name: troubleshoot-integration
description: "Use when: diagnosing Monaiq SDK integration issues, setup failures, authentication errors, license validation problems, checkout failures, or usage/consumption tracking bugs."
agent: monaiq
auto-invoke:
  - "User is having trouble integrating the Monaiq licensing SDK and sees errors or unexpected behavior"
  - "User encounters an error with licensing setup, authentication, or validation — not asking how to set up from scratch"
  - "User asks for help debugging a licensing integration issue — something was working or partially working and now isn't"
  - "User reports a specific error message, exception, or HTTP status code related to licensing"
tags: [integration, troubleshooting, diagnostics, errors]
category: integration
allowed-tools: [Read, Grep, Glob, register_or_login, profile, product, product_feature, offering, feature_offering, implement_base, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__offering, mcp__plugin_monaiq_monaiq__feature_offering, mcp__plugin_monaiq_monaiq__implement_base, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "errorCategory (setup|auth|validation|consumption)"
tier: 1
invoked-by: [user]
---

<objective>
Diagnose and resolve Monaiq SDK integration issues. Self-diagnose first from available context (error messages, code, configuration), then fall back to interactive diagnostic questions if the category or resolution can't be determined autonomously. Reference the troubleshooting resource as the diagnostic knowledge source.
</objective>

<output-context>
Provides to requesting context:
- diagnosis: { category: "setup" | "auth" | "validation" | "consumption", symptom, resolution, resolved: boolean }

This is a support entry point — users come here directly when something is broken.
</output-context>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only failure diagnosis, remediation options, and validation proof decisions.
</execution_context>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — fixes that change code/config require `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`).
</monaiq-agent-handoff>

<checkpoint-workflow-directive>
For `CHECKPOINT-APPLY-FIX`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, validation-remediation checkpoints, and any other hard checkpoint, follow `_shared/workflows/checkpoint.md` exactly. Use `checkpoint` in `monaiq_journal save_checkpoint` payloads, record the user's `result`, apply returned `.monaiq/*` file operations, and verify the checkpoint file exists before fixes continue.
</checkpoint-workflow-directive>

<workflow>
1. Run `_shared/workflows/startup.md` for `troubleshoot-integration`.
2. **Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **diagnosis + targeted fix**. Evidence sources, in priority order: error text + stack trace + HTTP status + validation command/result, affected code/config area in `targetProject`, recent journal decisions and `record_validation_failure` entries, profile/catalog/offering state, `monaiq://troubleshooting`, `monaiq://sdk/{stack}/setup`. Per-stack remediation shapes live in `monaiq://sdk/{stack}/setup` and `monaiq://troubleshooting` — do not embed framework-specific code in this skill source.
3. Fetch `monaiq://troubleshooting` before diagnosis. When the SDK stack is known, also fetch `monaiq://sdk/{stack}/setup`; supported concrete setup resources are `monaiq://sdk/dotnet/setup`, `monaiq://sdk/node/setup`, and `monaiq://sdk/react/setup`.
3. Capture the symptom from available evidence first: error text, stack trace, HTTP status, affected operation, relevant code/config area, recent journal decisions, profile/catalog/offering state, and validation command/result when available.
4. Classify the issue as setup, auth, validation, consumption, or checkout. Ask the user only when evidence cannot identify the category or missing input blocks diagnosis.
5. Present the diagnosis using `_shared/response-patterns.md` "Evidence Backing".
6. Follow the troubleshooting decision tree to form a likely root cause and proposed fix. If missing SDK/catalog/offering/profile/code evidence prevents a confident remediation recommendation, route to the appropriate prerequisite step before code/config/business-logic changes.
7. Gate fixes: read-only diagnostic recommendations use `CHECKPOINT-APPLY-FIX`; behavior-changing fixes use `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`. Record the user's result before edits.
8. Apply the smallest targeted fix, then verify with the relevant build, runtime, checkout, credential, or feature-check command. After validation failures, call `monaiq_journal record_validation_failure` with the observed error, validation command/result, likely cause, blocked next action, retry choices, and checkpoint requirement before further edits; use a failure-specific checkpoint before continuing remediation.
9. Coalesce useful error summaries, changed paths, validation proof, and handoff context through `_shared/workflows/completion.md`; use `record_file_changes` only when remediation changed workspace paths. Save `CHECKPOINT-SKILL-COMPLETE` with `proofOfDone` only when useful, apply returned operations, then call `skill_completed` once and hand off persisted `validationProof`.

When a setup or DI error indicates Monaiq guidance may be wrong for the installed SDK package/version, stop and record a plugin guidance defect with the package version, failing command, and compile/runtime error. Do not inspect DLLs, XML docs, NuGet caches, generated binaries, or local package folders to discover SDK signatures.
</workflow>

<reference>
## Self-Diagnose

Attempt to determine the error category and specific symptom automatically from:
- Error messages in the conversation (stack traces, HTTP status codes, exception types)
- Code context (if the user shared code snippets or files)
- Configuration context (if appsettings.json, .env, or package.json is visible)

Map to one of the 4 categories:

| Category | Signals |
|----------|---------|
| **Installation and setup issues** | Package install errors, NuGet/npm failures, missing config sections, DI registration exceptions |
| **Connection and credential problems** | 401/403 responses, "API key" mentions, connection refused, credential errors |
| **License check failures** | License not found, expired, revoked, wrong product, unexpected entitlement values |
| **Usage tracking and purchase flow issues** | Consumption reporting failures, checkout session errors, redirect issues |

If a specific symptom is identified, proceed directly to the matching decision tree from the troubleshooting resource and follow its diagnostic steps to resolution.

## Interactive Fallback

If the category cannot be determined automatically:
- Ask what the user is trying to do (install SDK, authenticate, validate license, report consumption, complete checkout)
- Ask for the specific error message or observed behavior
- Map responses to the appropriate error category and symptom

## Apply Decision Tree

Using the matching decision tree from the troubleshooting resource:
1. Present the diagnostic steps from the resource
2. Work through each step with the user
3. Identify the root cause
4. Present the resolution

Reference the troubleshooting resource content — do not duplicate it. Use its decision trees directly.

## Verify Resolution

After applying the resolution, confirm the issue is resolved:
- Ask the user to retry the operation
- If still failing, re-evaluate: check for a different root cause or escalate to a different error category

</reference>

<success_criteria>
- The troubleshooting resource was fetched and used as the diagnostic knowledge base
- Self-diagnosis was attempted first from available error context
- Interactive fallback was used only if self-diagnosis was insufficient
- All 4 error categories are coverable (setup, auth, validation, consumption/checkout)
- Resolution was applied and verified
</success_criteria>
