---
name: manage-catalog
description: Smart catalog builder — collapses product/feature/offering setup to 2 interactions with state detection and context flow
auto-invoke:
  - "User wants to create or manage their product catalog"
  - "User wants to set up products, features, and offerings"
  - "User asks how to define pricing tiers or feature assignments"
tags: [catalog, products, features, offerings, orchestration]
category: catalog
allowed-tools: [product, product_feature, offering, feature_offering]
argument-hint: ""
tier: 2
invoked-by: [getting-started]
disable-model-invocation: true
---

<input-context>
Receives from getting-started:
- userScenario: "greenfield" | "brownfield" | "returning" — determines workflow path
- detectedState: { hasProducts, hasOfferings, profileComplete, resellerEnabled } — skip completed steps
- quickStartSpec (optional): { appDescription, pricingChoice } — pre-filled from Quick Start mode

If invoked without upstream context (direct user request), run state detection to build equivalent state.
</input-context>

<output-context>
Provides to downstream skills:
- catalogSpec: { productCode, features: [{ key, type, displayName }], offerings: [{ code, classification, baseRate, currency, interval }], assignments: [{ featureKey, offeringCode, accessLevel | rateLimit }] }
- catalogComplete: true when product has features, offerings, and assignments verified

Chain: manage-catalog [catalogSpec] → implement-licensing [sdkConfig] → implement-feature
</output-context>

<state-detection>
Before executing any workflow, check current catalog state:
1. Call `product` (list) — check existing products
2. For each product, call `product_feature` (list) — check feature definitions
3. Call `offering` (list) — check existing offerings
4. For offerings with products, call `feature_offering` (list) — check assignments

Based on detected state, route to the appropriate step:
- No products → Start at Interaction 1 (full creation flow)
- Products exist, no features → Start at Interaction 1, skip product creation
- Products + features exist, no offerings → Start at Interaction 2
- Products + features + offerings exist, no assignments → Auto-create assignments
- Everything exists → Show catalog summary, offer modification options (utility workflows)
</state-detection>

<objective>
Build or modify a product catalog (products, features, pricing tiers, and feature assignments) through a streamlined 2-interaction flow with smart defaults, or manage existing catalog entities through utility workflows.
</objective>

<process>

## Prerequisites

- Active session — call `register_or_login` if not already authenticated
- Your account needs to be approved for catalog management (Monaiq calls this having `ResellerStatus = Enabled`) — check via the `profile` tool
- Fetch the domain model from `monaiq://domain/model` for entity context

## New Catalog Flow (Greenfield/Brownfield)

### Interaction 1: "What features does your product have?"

Ask the user to describe their product and its capabilities in plain language.

If `quickStartSpec.appDescription` was provided from getting-started, use it instead of asking.

From the user's description, infer:
- **Product**: name and code (derive code from name, e.g., "My Cool App" → `my-cool-app`)
- **Features**: map described capabilities to feature types:
  - Binary on/off capabilities → feature flags (called `ProductAccessFeature` in Monaiq)
  - Usage-counted capabilities → usage limits (called `ProductRateLimitFeature` in Monaiq)
  - Generate a feature key for each: lowercase-hyphenated from capability name (e.g., "Premium Dashboard" → `premium-dashboard`)

Present the inferred structure to the user for confirmation before creating.

Execute tool calls: `product` (create) → `product_feature` (create for each feature).

Smart defaults: Product Status = `Active`, feature key derived from name.

### Interaction 2: "How should they be priced?"

Ask how the user wants to charge, or use `quickStartSpec.pricingChoice` if provided.

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

Execute tool calls: `offering` (create for each tier) → `feature_offering` (assign features to each offering).

Default assignment values:
- Paid tiers: All feature flags (`ProductAccessFeature`) set to `Allowed`, usage limits (`ProductRateLimitFeature`) at user-specified or suggested quotas
- Free/Trial tiers: Feature flags selectively `Allowed`/`Denied` based on user description, usage limits at reduced quotas

**Read-back verification:** List offerings and feature assignments to confirm everything matches.

## Utility Workflows

When state detection shows everything already exists, offer these options:

- **Update features**: Add new features to existing product, modify feature key, change feature type
- **Modify offerings**: Change pricing, update billing interval, change license type (e.g., promote Trial to Subscription — called `LicenseClassification` in Monaiq)
- **Modify assignments**: Change feature values for a pricing tier — toggle access, adjust rate limits
- **Add tier**: Create a new pricing tier (Offering) and assign features (common when adding a free or enterprise tier)
- **View catalog**: Show full product → features → pricing tiers → feature assignments summary

</process>

<success_criteria>
- Product exists with features, offerings configured with pricing, all features assigned to offerings with correct values
- Read-back verification passes
- catalogSpec output-context populated for downstream skills
</success_criteria>
