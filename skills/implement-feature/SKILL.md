---
name: implement-feature
description: "Use when: adding Monaiq feature gates, access checks, premium feature enforcement, rate-limit assertions, consumption recording, or license feature checks to an SDK-integrated app."
agent: monaiq
auto-invoke:
  - "User wants to add feature gating to their application"
  - "User wants to implement license feature checks"
  - "User asks how to gate functionality by license features"
tags: [sdk, features, licensing, entitlements, access, ratelimit]
category: integration
allowed-tools: [Read, Write, Edit, Grep, Glob, Bash, product, product_feature, feature_offering, implement_product_feature, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__feature_offering, mcp__plugin_monaiq_monaiq__implement_product_feature, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "featureKey, featureType (access|ratelimit)"
tier: 3
invoked-by: [implement-licensing, manage-catalog]
---

<objective>
Add license feature checks to an SDK-integrated .NET or React app without guessing SDK APIs or hiding premium capability state. The skill handles feature discovery, checkpointed implementation planning, authoritative tool guidance, build validation, and journal completion.
</objective>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — business-logic edits require `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`).
</monaiq-agent-handoff>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only feature-gate, rate-limit, consumption, and validation decisions.
</execution_context>

<input-output-contract>
Input from `implement-licensing`: `sdkConfig` plus optional `targetFeatures`. Direct invocation must recreate that context by checking SDK setup and catalog feature state.

Output: `featureImpl: { implementedFeatures: [{ featureKey, featureType, codeFile }], buildVerified: boolean }`.
</input-output-contract>

<tool-first-authority>
implement_product_feature is authoritative for feature-gate implementation. Use `implement_product_feature` and `fetch_step_resources` as the normal implementation guidance paths before code/config/business-logic changes.

SDK reverse-engineering is a plugin guidance defect. Do not inspect DLLs. Do not inspect XML documentation. Do not inspect NuGet package caches. Do not inspect generated binaries to discover SDK signatures, namespaces, assertion types, feature records, or runtime behavior.

If Monaiq tool or resource guidance is missing, contradictory, or insufficient, stop before code/config/business-logic changes and record a plugin guidance defect with monaiq_journal record_error.
</tool-first-authority>

<checkpoint-workflow-directive>
For `CHECKPOINT-FEATURE-SELECTION`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, and any other hard checkpoint, follow `_shared/workflows/checkpoint.md` exactly. Use `checkpoint` in `monaiq_journal save_checkpoint` payloads, record the user's `result`, apply returned `.monaiq/*` file operations, and verify the checkpoint file exists before business-logic edits continue.
</checkpoint-workflow-directive>

<workflow>
1. Run `_shared/workflows/startup.md` for `implement-feature`.
2. **Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **feature implementation (business-logic edits)**. Evidence sources, in priority order: selected feature from route packet/journal, existing UI/business-logic locations in `targetProject`, SDK integration state from `implement-licensing`, catalog/offering facts, journal decisions, `monaiq://platforms/api-surface/{platform}`, `monaiq://docs/anti-patterns/{platform}`, `monaiq://platforms/pitfalls/{platform}`. Per-stack code shapes live in `monaiq://sdk/{stack}/setup` and the `implement_product_feature` server response — do not embed framework-specific code in this skill source.
3. Resume through `CHECKPOINT-RESUME` when the state packet is current; call additional journal startup actions only when the shared startup workflow reports missing or stale state.
3. Read the master journey checklist from `.monaiq/STATE.md`; this skill owns only the feature implementation gate. Establish prerequisites in this order: SDK integration present, product features available, platform resources fetched, existing feature checks scanned. Missing SDK routes to `implement-licensing`; missing catalog features route to `manage-catalog`.
4. Fetch required resources before implementation: `monaiq://domain/model`, `monaiq://domain/namespaces`, `monaiq://platforms/api-surface/{platform}`, `monaiq://docs/anti-patterns/{platform}`, `monaiq://sdk/{stack}/setup`, and `monaiq://platforms/pitfalls/{platform}`.
5. Use evidence before asking a new question. Infer the feature path from the selected feature, route packet, existing UI/business-logic location, SDK state, catalog/offering facts, and journal decisions. You must confirm inferred decisions in the next existing checkpoint, especially `CHECKPOINT-FEATURE-SELECTION` or `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, with labeled assumptions for credential handling impact, checkout architecture impact, feature path, and next steps. Do not add a new checkpoint name solely for evidence inference.
6. Present the plan using `_shared/response-patterns.md` "Evidence Backing" plus selected feature, feature kind, source files/areas, and validation plan.
7. Call `implement_product_feature` with `startStep=all` when context is sufficient; use `startStep=1` then `startStep=2` only when step-by-step review improves safety.
8. Stop at `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT` before adding or changing feature gates, access checks, rate-limit assertions, consumption recording, UI locked states, or other business logic. Record the user's approval result before edits.
9. Apply code changes using the authoritative tool/resource guidance only. If guidance is missing, contradictory, or insufficient, stop and record a plugin guidance defect with `monaiq_journal record_error`.
10. Build and exercise allow, denied, expired/misconfigured, and over-limit paths where applicable. Record validation failures with `monaiq_journal record_validation_failure` before remediation.
11. Coalesce changed paths, validation proof, feature implementation checklist progress, and `featureImpl` handoff through `_shared/workflows/completion.md`; use `record_file_changes` only when changed paths exist. Call `update_checklist_progress` for feature implementation only after source skill, relevant MCP tool, canonical resources, and checkpoint/journal evidence prove the gate is complete before marking the checklist gate complete. Save `CHECKPOINT-SKILL-COMPLETE` with `proofOfDone` only when useful, apply returned file operations using the **File Operation Application Protocol** in `_shared/protocols.md`, then call `skill_completed` once and hand off persisted `featureImpl` or `validationProof`.
</workflow>

<experience-contract>
## Locked Feature Experience Contract

Keep premium capabilities visible, valuable, and safely locked. A denied user should understand what the capability does, why it is unavailable, and which unlock or recovery path fits the current task before protected execution fails. Do not hide premium affordances when showing a visible locked state would improve product understanding.

backend/business-logic enforcement is authoritative. frontend checks are for frontend state communication: loading, available, locked, expired, over-limit, and misconfigured states. Reuse the existing provider, authorization cache, and state snapshot rather than duplicating authorization calls across every component. Backend/business-logic enforcement must still guard the protected action.

locked, expired, over-limit, and misconfigured outcomes need distinct user messages and actions. Locked means upgrade or activate; expired means renew or contact the buyer; over-limit means retry after reset or upgrade; misconfigured means recover setup or contact support. Authorization and rate-limit exceptions are product and diagnostic signals and must not be silently swallowed; preserve them in logs, journals, telemetry, or validation output before translating them to user-facing recovery.

Rate-limit experiences must treat unlimited and capped as explicit states. For capped usage, show current usage, remaining quota, reset cadence, meaningful warning threshold guidance before exhaustion, deterministic denial at the limit, and a recovery path such as wait for reset, retry later, or upgrade. For unlimited usage, say unlimited rather than implying a hidden quota.

Use host-native UI quality: compact, accessible, responsive, aligned with the source app's component system, and persuasive without becoming marketing-heavy. Before changing business logic or UI, use `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT` and summarize affected pages, components, flows, and user-visible impact.

</experience-contract>

<reference>
## Feature Discovery Reference

Identify the feature to gate and determine its type. Use the `product_feature` tool to list features for a product. Each feature exposes its key, kind, display name, and access-specific service type. Each feature type has its own assertion pattern: Access features are binary allowed/denied gates; RateLimit features require feature retrieval, consumption recording, and assertion. Resolve `monaiq://platforms/api-surface/{platform}` and `monaiq://domain/model` for exact platform types, enum values, and call sequence.

## Feature-Type Decision Table

| Aspect | Feature Flag (Access) | Usage Limit (RateLimit) |
|--------|----------------------|-------------------------|
| Pattern | Assert → allowed/denied | Get feature → record consumption → assert |
| Use case | Premium content, feature flags, capability toggles | API rate limits, usage quotas, metered operations |
| Cardinality | One assertion per check | Multi-step: retrieve → record → assert |

## Implementation Reference

- Place all licensing-related string literals (feature keys, offering ids, redirect URLs) in a single constants module (e.g. `LicensingConstants.cs` for .NET, `licensingConstants.ts` for React). Reference them by symbol — do NOT inline the literal in business logic. This is mandatory (Phase 14 D-30).

### Feature Flag Check (Access — Binary Gate)

Use when: a binary "allowed / not allowed" decision is sufficient (premium gate, capability toggle, feature-flag rollout).

Resolve platform-specific assertion types, call sequence, and runtime wiring through the required resources listed in the workflow before editing code.

### Usage Limit Check (RateLimit — Metered Consumption)

Use when: usage must be metered and bounded (API rate limits, per-period quotas, consumption-based features). Pattern is always multi-step — retrieve the feature record, record consumption, then assert whether the limit is still respected.

<!-- SEM-01-stopgap -->
> **Unlimited assignments.** In the domain model, `RateLimit = 0` represents an unlimited entitlement. When creating or updating a rate-limit assignment through the `feature_offering` MCP tool, send both `RateLimit` and `SampleSeconds` as the string `"unlimited"`; the tool maps that to the internal zero representation. Non-zero capped values must be positive integers for both fields.
<!-- /SEM-01-stopgap -->

Rate-limit and consumption exceptions are product signals and must not be silently swallowed. Surface them in logs/UI/validation paths so the user understands when an entitlement or quota blocked the action.

**Error-boundary pattern for rate-limit violations** (consumer-built — Monaiq ships
the typed `RateLimitError`, you build the UI):

```tsx
class RateLimitBoundary extends React.Component<{children: React.ReactNode}, {hit: boolean}> {
  state = { hit: false };
  static getDerivedStateFromError(e: unknown) { return { hit: e instanceof RateLimitError }; }
  render() { return this.state.hit ? <p>Rate limited — try again soon.</p> : this.props.children; }
}
```

### Build and Verify

Build-and-verify guidance is delivered by `implement_product_feature` step 2 via its `hints` array. Follow those hints after implementing the feature check.

</reference>

<related>
## Related Tools

- `implement_product_feature` — Interactive step-by-step feature integration (call `startStep=1` then `startStep=2` consecutively).
- `product` — List products in the catalog.
- `product_feature` — List features for a product.
- `implement_base` — SDK integration (prerequisite).

## Related Resources

- `monaiq://domain/model` — Entity relationships and feature-type hierarchy.
- `monaiq://domain/namespaces` — Authoritative namespace-to-type mappings.
- `monaiq://platforms/api-surface/dotnet` — .NET assertion types, method signatures, semantics.
- `monaiq://platforms/api-surface/react` — React assertion types, hook signatures, semantics.
- `monaiq://platforms/pitfalls/dotnet` — .NET known issues.
- `monaiq://platforms/pitfalls/react` — React known issues (provider remount, async hook reuse).
- `monaiq://docs/anti-patterns/dotnet` — .NET anti-patterns for license checks and rate-limit enforcement.
- `monaiq://docs/anti-patterns/react` — React anti-patterns for license checks and rate-limit enforcement.
- `monaiq://sdk/dotnet/setup` — .NET SDK runtime wiring narrative.
- `monaiq://sdk/react/setup` — React SDK runtime wiring narrative.
</related>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Feature not found in catalog | `product_feature` list returns empty or missing feature | Verify the product code is correct. Route to `manage-catalog` to create missing features. |
| Wrong assertion type used | Runtime type-mismatch error | Check the feature's kind — resolve `monaiq://platforms/api-surface/{platform}` for the correct assertion type per kind, and `monaiq://domain/model` for the kind enum. |
| Namespace import errors | Build fails with missing type references | Resolve `monaiq://domain/namespaces` and verify all imports match the authoritative reference. |
| Assertion returns unexpected result | Feature check returns denied when it should be allowed | Verify the feature is assigned to the user's offering with the correct value (Allowed, not Denied). Check via the `feature_offering` tool. |
| Consumption recording fails | Recording operation throws for RateLimit features | Verify the feature type is RateLimit (not Access). Check that the consumption amount is a positive number. Resolve `monaiq://platforms/pitfalls/{platform}` for platform-specific error modes. |

Feature-gate code is additive — failed implementations can be corrected by editing the source file. No rollback needed.
</error-recovery>

<success_criteria>
- Correct assertion type used for each feature kind (Access vs. RateLimit).
- Feature checks return allowed for valid licenses with the feature granted.
- Application handles the denied case gracefully.
- No namespace errors — all imports match `monaiq://domain/namespaces`.
- Project builds without errors.
</success_criteria>
