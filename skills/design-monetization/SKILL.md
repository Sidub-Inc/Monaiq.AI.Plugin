---
name: design-monetization
description: "Use when: designing pricing strategy, tiers, subscriptions, trials, perpetual licenses, usage-based billing, or catalog-ready offering plans; does not create catalog entities."
agent: monaiq
auto-invoke:
  - "User wants to design pricing tiers or a monetization strategy for their product"
  - "User asks 'how should I price this' or 'subscription vs perpetual' or 'what pricing model should I use'"
  - "User wants help structuring pricing plans, billing intervals, or tier definitions"
  - "User mentions pricing confusion, pricing strategy, or monetization approach"
tags: [strategy, pricing, monetization, tiers, design]
category: strategy
allowed-tools: [register_or_login, product, product_feature, offering, feature_offering, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__offering, mcp__plugin_monaiq_monaiq__feature_offering, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
tier: 1
invoked-by: [user, analyze-codebase, getting-started]
---

<objective>
Design catalog-ready pricing tiers from scenario, capability, codebase, catalog, and journal evidence. This skill recommends and refines monetization strategy; it does not create or mutate catalog entities.
</objective>

<input-context>
Receives from scenario-advisor:
- selectedScenario (optional): { name, model, featureTypes, offeringModel } — chosen licensing model
- appType (optional): application type for market norms

Also accepts from analyze-codebase (if chained directly):
- capabilities (optional): identified features to price

If invoked without upstream context, gathers requirements through conversation.
</input-context>

<output-context>
Provides to manage-catalog:
- pricingPlan: { tiers: [{ name, classification, baseRate, currency, interval, features: [{ key, type, value }] }] }
- productName: user's product name

Discovery chain endpoint: design-monetization [pricingPlan] → manage-catalog (creates catalog entities from the plan)
</output-context>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only catalog-ready pricing strategy and feature-assignment recommendations; catalog mutation belongs to `manage-catalog`.
</execution_context>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — `CHECKPOINT-PRICING-APPROVAL` is a hard checkpoint).
</monaiq-agent-handoff>

<workflow>
1. Run `_shared/workflows/startup.md` for `design-monetization`.
2. Resolve `monaiq://patterns/pricing` and `monaiq://domain/model` before proposing tiers. Stop before pricing recommendations if pricing or entity context is unavailable.
3. **Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **pricing approval**. Evidence sources, in priority order: `selectedScenario` from `scenario-advisor`, capability evidence from `analyze-codebase`, existing offerings (`offering` list when catalog exists), journal decisions, `monaiq://patterns/pricing`, conversation cues. If a codebase is reachable but no analysis exists, route to `analyze-codebase` first.
4. Propose a complete tier structure up front using fetched pricing patterns and the evidence cite. Mark one structure as the recommendation; alternatives appear only when they resolve real ambiguity. The host-native question lets the user approve, refine tiers, swap structure, or pause to inspect pricing patterns.
5. When existing offerings are present, treat them as evidence for refinement rather than overwriting them.
6. Use `CHECKPOINT-PRICING-APPROVAL` before treating pricing, tier, or feature-assignment strategy as approved for catalog creation.
7. Output a catalog-ready `pricingPlan` for `manage-catalog`, coalesce pricing decisions + checklist progress + handoff context into the completion workflow, then call `skill_completed` once.
</workflow>

<reference>
## State Routing Reference

- Scenario provided, no existing offerings -> use scenario to inform tier design.
- No upstream context -> gather product, customer, and pricing-model constraints through conversation.
- Existing offerings -> show current pricing and offer refinement.

## Gather Context

Determine design inputs from available context:

- If `scenario-advisor` was run: use the selected scenario (app type, chosen model, feature types) to inform tier structure
- If `analyze-codebase` was run: use identified capabilities and their classifications
- If neither: gather from conversation — ask what the product does, who the customers are, and what pricing model they want

Infer the appropriate number of tiers from: scenario-advisor output (if available), application complexity, identified feature count, and market norms for the application type.

## Propose Tier Structure

Using pricing patterns from `monaiq://patterns/pricing`, propose a complete tier structure:

| Tier | Name | License Type | Price | Interval | Features Included |
|------|------|-------------|-------|----------|-------------------|
| 1 | [e.g., Free] | Subscription | $0 | 1 Month | [list with access/rate-limit values] |
| 2 | [e.g., Pro] | Subscription | $X | 1 Month | [list with access/rate-limit values] |
| 3 | [e.g., Enterprise] | Subscription | $Y | 1 Month | [list with access/rate-limit values] |

For each feature in each tier, specify:
- Feature flags (Access features): `Allowed` or `Denied`
- Usage limits (RateLimit features): quota value and time window (e.g., "1000 per hour")

<!-- SEM-01-stopgap -->
> **Unlimited tiers.** When calling the `feature_offering` MCP tool, set both `RateLimit` and `SampleSeconds` to the string `"unlimited"` for an uncapped assignment. The tool maps that input to the domain's zero-as-unlimited representation internally. Paid capped tiers set explicit positive values (e.g. `RateLimit = 1000` with `SampleSeconds = 3600` for 1000/hour).
<!-- /SEM-01-stopgap -->

Explain the rationale: why this many tiers, why these price points relative to each other, and which pricing pattern from `monaiq://patterns/pricing` this follows.

## Refine with User

Present the proposal and invite feedback. Common refinements:
- Adjust tier count (add/remove tiers)
- Change feature inclusion (move features between tiers)
- Modify pricing (change base rates, intervals)
- Change license type (switch from Subscription to Perpetual — called `LicenseClassification` in Monaiq)
- Add consumption metering (layer on usage limits)

Iterate until the user approves the tier structure.

## Output Catalog-Ready Summary

Produce a final structured summary table ready for `manage-catalog` input:

| Tier Name | Code | License Type | Price | Currency | Interval | Included Features (FeatureKey: Value) |
|-----------|------|-------------|-------|----------|----------|--------------------------------------|
| [name] | [code] | [type] | [price] | [currency] | [interval] | [feature1: Allowed, feature2: 500/hr] |

This table contains all fields needed by the `manage-catalog` skill to create the catalog: product code, offering codes, feature keys with assignment values.

## Route to Catalog

Suggest: "To create this catalog in Monaiq, run the `manage-catalog` skill with the specification above."

</reference>

<success_criteria>
- `monaiq://patterns/pricing` was fetched and used for pattern knowledge
- Tier count was inferred from context, not a fixed default
- Complete tier structure was proposed up front
- User refined and approved the design
- Final output is a catalog-ready summary table with all manage-catalog input fields
- `manage-catalog` suggested as next step
</success_criteria>
