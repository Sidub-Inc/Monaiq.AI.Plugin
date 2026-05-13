---
name: manage-catalog
description: "Use when: creating or managing a Monaiq product catalog, products, features, offerings, pricing tiers, and feature assignments after onboarding or monetization design."
agent: monaiq
auto-invoke:
  - "User wants to create or manage their product catalog"
  - "User wants to set up products, features, and offerings"
  - "User asks how to define pricing tiers or feature assignments"
tags: [catalog, products, features, offerings, orchestration]
category: catalog
allowed-tools: [register_or_login, profile, product, product_feature, offering, feature_offering, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__offering, mcp__plugin_monaiq_monaiq__feature_offering, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
tier: 2
invoked-by: [getting-started]
---

<objective>
Create or update a Monaiq product catalog through evidence-backed recommendation, checkpointed mutation, and read-back verification. The skill owns product, feature, offering, and feature-assignment tool calls; it does not own SDK setup or app code changes.
</objective>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — catalog mutations require hard checkpoints).
</monaiq-agent-handoff>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only catalog-specific decisions and `product`/`product_feature`/`offering`/`feature_offering` tool calls.
</execution_context>

<input-output-contract>
Input from `getting-started`: `userScenario`, `detectedState`, optional `quickStartSpec`, and route boundaries (`activePlatform`, `targetProject`, `outOfScopePlatforms`). Direct invocation must rebuild equivalent context through session/profile/catalog state detection.

Output: `catalogSpec: { productCode, features: [{ key, type, displayName }], offerings: [{ code, classification, baseRate, currency, interval }], assignments: [{ featureKey, offeringCode, accessLevel | rateLimit }] }`, preserved route boundaries (`activePlatform`, `targetProject`, `outOfScopePlatforms`), plus `catalogComplete: true` after read-back verification. Downstream chain: `manage-catalog` -> `implement-licensing` -> `implement-feature`.
</input-output-contract>

<checkpoint-workflow-directive>
For `CHECKPOINT-PRE-CATALOG-MUTATION`, `CHECKPOINT-PRE-PUBLISH-OFFERING`, and any other hard checkpoint, follow `_shared/workflows/checkpoint.md` exactly. Use `checkpoint` in `monaiq_journal save_checkpoint` payloads, record the user's `result`, apply returned `.monaiq/*` file operations, and verify the checkpoint file exists before mutation continues.
</checkpoint-workflow-directive>

<workflow>
1. Run `_shared/workflows/startup.md` for `manage-catalog`.
2. Read the master journey checklist from `.monaiq/STATE.md`; this skill owns only the catalog and offerings gate. Establish prerequisites: active session, `ResellerStatus = Enabled`, and `monaiq://domain/model` fetched. Stop before catalog mutation when session/profile/domain evidence is missing.
3. Detect backend catalog state in order: `product` list, `product_feature` list for relevant products, `offering` list, and `feature_offering` list for offerings. Backend catalog state wins; journal records decisions and outcomes only.
4. Use evidence before asking a new question. Infer catalog recommendation next steps from codebase evidence, route packet context, existing product/offering facts, and journal decisions. You must confirm inferred decisions in the next existing checkpoint, especially `CHECKPOINT-PRE-CATALOG-MUTATION`, with labeled assumptions for product structure, feature path, offering shape, credential handling impact, checkout architecture impact, and next steps. Do not add a new checkpoint name solely for evidence inference.
5. Present the recommendation using `_shared/response-patterns.md` "Evidence Backing" plus the exact `product`, `product_feature`, `offering`, or `feature_offering` operations proposed. Every product, feature, offering, pricing, or assignment proposal is an evidence-backed catalog recommendation.
6. Stop at `CHECKPOINT-PRE-CATALOG-MUTATION` before any `product`, `product_feature`, `offering`, or `feature_offering` create/update/delete call. Record the user's result before tool calls run.
7. Execute catalog mutations in dependency order: product, features, offerings, feature assignments. Keep offerings in Draft unless the user explicitly approves publication through `CHECKPOINT-PRE-PUBLISH-OFFERING`.
8. Perform mandatory read-back verification after create/update of paid-tier assignments: list `feature_offering` rows for each paid offering, verify intended `ServiceAccessLevel` or rate-limit values, and stop on mismatches before claiming catalog completion.
9. Coalesce changed catalog entities, read-back proof, checklist progress, preserved route boundaries, and `catalogSpec` handoff through `_shared/workflows/completion.md`. Call `update_checklist_progress` for catalog and offerings only after source skill, relevant MCP tool, canonical resources, and checkpoint/journal evidence prove the gate is complete before marking the checklist gate complete. Save `CHECKPOINT-SKILL-COMPLETE` with `proofOfDone` only when useful, apply returned operations using the **File Operation Application Protocol** in `_shared/protocols.md`, call `skill_completed` once, and hand off persisted `catalogSpec` plus `activePlatform`, `targetProject`, and `outOfScopePlatforms` to `implement-licensing`. Emit the **Next Step Signal**: `**Next:** Invoke \`implement-licensing\` — integrate the Monaiq SDK into your application to enforce licensing.`
</workflow>

<reference>
## State Routing Reference

- No products -> full creation flow.
- Products exist, no features -> create features for existing product.
- Products and features exist, no offerings -> create offerings.
- Products, features, offerings exist but assignments are missing -> create assignments.
- Everything exists -> show catalog summary and use utility workflows.

## New Catalog Flow

**Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **catalog mutation**. Evidence sources, in priority order: `quickStartSpec` from `getting-started`, `pricingPlan` from Discovery delegation (`analyze-codebase` + `scenario-advisor` + `design-monetization`), prior journal `discoveredCapabilities`, existing catalog state, conversation cues. Recommendation precedes evidence in every reply; never open with *"what features does your product have?"*.

### Interaction 1: starter product and features

Recommend a starter product (name + derived code) and feature set inferred from upstream evidence. Map binary capabilities to `ProductAccessFeature`, usage-counted capabilities to `ProductRateLimitFeature`; derive feature keys lowercase-hyphenated from the capability name. Present as the default; invoke the **Host-Native Ask Pattern** (see `_shared/protocols.md` § Host-Native Ask Pattern) with options: approve the starter catalog, refine features, refine the product name, or pause to inspect the source analysis. Block until the user responds.

Confirm via `CHECKPOINT-PRE-CATALOG-MUTATION` before any `product` / `product_feature` create call. Smart defaults: Product Status = `Active`, feature key derived from name.

### Interaction 2: starter pricing

Ask how the user wants to charge, or use `quickStartSpec.pricingChoice` if provided. When asking, invoke the **Host-Native Ask Pattern** (see `_shared/protocols.md` § Host-Native Ask Pattern) with the pricing options listed below as option labels. Block until the user responds.

Present pricing options in plain language:
- **"Free trial + paid subscription"** → Create a Trial offering (14 days, $0) + Subscription offering ($X/month)
- **"Just a subscription"** → Create a Subscription offering ($X/month)
- **"One-time purchase"** → Create a Perpetual offering ($X one-time)
- **"Free tier + paid tier"** → Create a free Subscription ($0/month) + paid Subscription ($X/month)
- **"Usage-based"** → Create a Subscription with usage limit values scaled per tier

For each pricing tier (called an Offering in Monaiq), apply smart defaults:
- Currency: `USD`
- Interval: 1 Month for Subscription, none for Perpetual, 14 days for Trial
- Status: `Draft` (safe default — user publishes when ready)

[CONFIRM] Execute tool calls: `offering` (create for each tier) → `feature_offering` (assign features to each offering).

Default assignment values:
- Paid tiers: All feature flags (`ProductAccessFeature`) set to `Allowed`, usage limits (`ProductRateLimitFeature`) at user-specified or suggested quotas
- Free/Trial tiers: Feature flags selectively `Allowed`/`Denied` based on user description, usage limits at reduced quotas

- Catalog identifiers (offering ids, product ids, feature keys) referenced in code MUST live in a constants module. Inline string literals are forbidden (D-30).

**Required payload shape — `feature_offering` create for an access feature.** `ServiceAccessLevel` is required; omitting it is rejected by the API. Always send it explicitly:

```json
{
  "__sidub_entityType": "ProductFeatureOffering.ServiceAccess",
  "OfferingId": "<offering-id>",
  "FeatureKey": "<feature-key>",
  "ServiceAccessLevel": "Allowed"
}
```

For rate-limit features, both `SampleSeconds` and `RateLimit` are required (use matching `"unlimited"` strings or matching positive integers — asymmetric combinations are rejected).

**Read-back verification (mandatory after any create/update of paid-tier assignments):**
1. Call `feature_offering` (list) for each paid offering.
2. Assert every `ProductFeatureOffering.ServiceAccess` row has `ServiceAccessLevel: "Allowed"` (unless the user explicitly intended `Denied` for that tier).
3. If any assignment shows `ServiceAccessLevel: "Denied"` that the user did not intend, stop and surface it to the user — this is the silent-failure mode where customers receive credentials but `AssertLicense` always returns `false`. Re-run `feature_offering` (update) with the correct level.

## Utility Workflows

When state detection shows everything already exists, offer these options:

- **Update features**: Add new features to existing product, modify feature key, change feature type
- **Modify offerings**: Change pricing, update billing interval, change license type (e.g., promote Trial to Subscription — called `LicenseClassification` in Monaiq)
- **Modify assignments**: Change feature values for a pricing tier — toggle access, adjust rate limits
- **Add tier**: Create a new pricing tier (Offering) and assign features (common when adding a free or enterprise tier)
- **View catalog**: Show full product → features → pricing tiers → feature assignments summary

</reference>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Product creation fails | Tool returns error on `product` create | Retry once. If persistent, check session is authenticated via `register_or_login`. |
| Feature creation fails after product created | `product_feature` returns validation error | Product is safe — retry the feature creation with corrected input. List existing features first to avoid duplicates. |
| Offering creation fails after features created | `offering` returns error | Product and features are safe. Retry offering creation. Check that LicenseClassification is valid (Trial, Subscription, or Perpetual). |
| Feature assignment fails after offering created | `feature_offering` returns error | Offering exists but is incomplete. Retry assignment. Verify FeatureKey matches exactly and feature type matches assignment type (Access → ServiceAccessFeatureOffering, RateLimit → RateLimitFeatureOffering). |
| Partial catalog state | Some entities created, process interrupted | Re-run state detection — the skill will resume from the last incomplete step. Already-created entities are preserved. |

All catalog operations are idempotent at the entity level — creating a product that already exists will return the existing entity. Feature assignments can be updated in place.
</error-recovery>

<success_criteria>
- Product exists with features, offerings configured with pricing, all features assigned to offerings with correct values
- Read-back verification passes
- catalogSpec output-context populated for downstream skills
</success_criteria>
