---
name: design-monetization
description: Pricing tier designer with context flow — proposes a pricing structure based on your features and licensing model, outputs a catalog-ready plan
auto-invoke:
  - "User wants to design pricing tiers for their product"
  - "User wants to create a monetization strategy"
  - "User asks how to structure their pricing plans"
tags: [strategy, pricing, monetization, tiers, design]
category: strategy
allowed-tools: []
argument-hint: ""
tier: 3
invoked-by: [analyze-codebase, getting-started]
---

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

<state-detection>
Before designing tiers:
1. Check if a selectedScenario was provided from scenario-advisor
2. Call `offering` (list) — check if pricing tiers (Offerings) already exist

Based on state:
- Scenario provided, no existing offerings → Use scenario to inform tier design (Step 2)
- No upstream context → Gather from conversation (Step 1)
- Existing offerings → Show current pricing, offer refinement (adjust tiers, add/remove features)
</state-detection>

<objective>
Guide an agent through designing a complete pricing tier structure for a user's product. Propose a tier structure up front based on context from `scenario-advisor` (or conversation), then refine through user feedback. Output a catalog-ready summary table that feeds directly into the `manage-catalog` skill.
</objective>

<process>

## Prerequisites

- Fetch `monaiq://patterns/pricing` for pricing pattern knowledge (base billing models, consumption overlay, industry compositions)
- Fetch `monaiq://domain/model` for entity context (ProductOffering fields, FeatureOffering types)

## Step 1: Gather Context

Determine design inputs from available context:

- If `scenario-advisor` was run: use the selected scenario (app type, chosen model, feature types) to inform tier structure
- If `analyze-codebase` was run: use identified capabilities and their classifications
- If neither: gather from conversation — ask what the product does, who the customers are, and what pricing model they want

Infer the appropriate number of tiers from: scenario-advisor output (if available), application complexity, identified feature count, and market norms for the application type.

## Step 2: Propose Tier Structure

Using pricing patterns from `monaiq://patterns/pricing`, propose a complete tier structure:

| Tier | Name | License Type | Price | Interval | Features Included |
|------|------|-------------|-------|----------|-------------------|
| 1 | [e.g., Free] | Subscription | $0 | 1 Month | [list with access/rate-limit values] |
| 2 | [e.g., Pro] | Subscription | $X | 1 Month | [list with access/rate-limit values] |
| 3 | [e.g., Enterprise] | Subscription | $Y | 1 Month | [list with access/rate-limit values] |

For each feature in each tier, specify:
- Feature flags (Access features): `Allowed` or `Denied`
- Usage limits (RateLimit features): quota value and time window (e.g., "1000 per hour")

Explain the rationale: why this many tiers, why these price points relative to each other, and which pricing pattern from `monaiq://patterns/pricing` this follows.

## Step 3: Refine with User

Present the proposal and invite feedback. Common refinements:
- Adjust tier count (add/remove tiers)
- Change feature inclusion (move features between tiers)
- Modify pricing (change base rates, intervals)
- Change license type (switch from Subscription to Perpetual — called `LicenseClassification` in Monaiq)
- Add consumption metering (layer on usage limits)

Iterate until the user approves the tier structure.

## Step 4: Output Catalog-Ready Summary

Produce a final structured summary table ready for `manage-catalog` input:

| Tier Name | Code | License Type | Price | Currency | Interval | Included Features (FeatureKey: Value) |
|-----------|------|-------------|-------|----------|----------|--------------------------------------|
| [name] | [code] | [type] | [price] | [currency] | [interval] | [feature1: Allowed, feature2: 500/hr] |

This table contains all fields needed by the `manage-catalog` skill to create the catalog: product code, offering codes, feature keys with assignment values.

## Step 5: Route to Catalog

Suggest: "To create this catalog in Monaiq, run the `manage-catalog` skill with the specification above."

</process>

<success_criteria>
- `monaiq://patterns/pricing` was fetched and used for pattern knowledge
- Tier count was inferred from context, not a fixed default
- Complete tier structure was proposed up front
- User refined and approved the design
- Final output is a catalog-ready summary table with all manage-catalog input fields
- `manage-catalog` suggested as next step
</success_criteria>
