---
name: getting-started
description: Get started with Monaiq — create an account, set up your first product, and choose your next step in under 5 minutes
auto-invoke:
  - "User wants to get started with Monaiq licensing and has not yet authenticated or set up products"
  - "User asks what Monaiq can do for them and needs an overview, not a specific task"
  - "User is new to Monaiq and needs orientation — no existing products or SDK integration"
  - "User asks 'where do I start' or 'how do I begin' with licensing"
tags: [onboarding, getting-started, orchestration, state-detection]
category: onboarding
allowed-tools: [register_or_login, getting_started, profile, product, offering, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__getting_started, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__offering]
tier: 1
invoked-by: [user]
---

<objective>
State-aware onboarding that detects user progress, classifies their scenario (greenfield, brownfield, or returning), and routes intelligently to the next action. Supports Quick Start mode for greenfield users to collapse the full journey into 2 interactions.
</objective>

<glossary-for-humans>

When users describe what they want using everyday language, map their terms to Monaiq domain concepts:

| User Says | Monaiq Concept | Details |
|-----------|---------------|---------|
| "subscription" / "recurring billing" | ProductOffering | LicenseClassification = Subscription |
| "free trial" / "trial period" | ProductOffering | LicenseClassification = Trial |
| "one-time purchase" / "perpetual license" | ProductOffering | LicenseClassification = Perpetual |
| "license key" / "activation key" | EncodedCredential | Retrieved via the `profile` tool |
| "feature flag" / "premium feature" | ProductAccessFeature | Gated by FeatureKey |
| "usage limit" / "API quota" / "rate limit" | ProductRateLimitFeature | Metered by FeatureKey |
| "pricing tier" / "plan" | ProductOffering | Bundles features at a price point |
| "product" / "app" / "software" | Product | The scope boundary for all features and offerings |

Use this glossary when interpreting user requests throughout the onboarding flow. Translate user vocabulary into domain terms before calling tools.

</glossary-for-humans>

<output-context>
Provides to downstream skills:
- userScenario: "greenfield" | "brownfield" | "returning" — classified in Step 3
- detectedState: { hasProducts: boolean, hasOfferings: boolean, profileComplete: boolean, resellerEnabled: boolean }
- quickStartSpec (optional, greenfield only): { appDescription: string, pricingChoice: string } — from Quick Start Step 4

Routes to:
- manage-catalog: receives { userScenario, detectedState, quickStartSpec? }
- implement-licensing: receives { userScenario, detectedState }
- analyze-codebase: receives { userScenario, detectedState }
</output-context>

<process>

## Step 1: Authentication & User Detection

Call the `register_or_login` tool to authenticate and establish a session.

**If the tool indicates no existing account — this is a NEW user:**
- Guide through registration: "Provide your email to create a Monaiq account."
- After registration completes, proceed to Step 2.

**If login succeeds — this is a RETURNING user:**
- Proceed to Step 2 with returning-user context.

## Step 2: State Detection

Gather the user's current state by calling these tools in sequence:

1. Call `getting_started` to get the onboarding checklist — shows which setup steps are complete.
2. Call `profile` to get `ProfileStatus`, `ResellerStatus`, and `IssuerClientId`.
3. Call `product` with a list action to check for existing products.
4. If products exist, call `offering` with a list action to check for existing offerings.

Build a state picture from the results:

| State Variable | How to Determine |
|---------------|-----------------|
| `hasProducts` | Product list is non-empty |
| `hasOfferings` | Offering list is non-empty |
| `profileComplete` | ProfileStatus is "Completed" |
| `resellerEnabled` | ResellerStatus is "Enabled" |

## Step 3: Scenario Classification

Based on the detected state, classify the user into one of three scenarios:

### Scenario A — Greenfield (new user, no catalog)

**Conditions:** No products AND no offerings (OR new registration from Step 1).

Present two options:
- **Quick Start** (recommended for new users) — collapses the journey to 2 interactions. See Step 4.
- **Full Guided Flow** — step-by-step setup:
  - If profile is incomplete → route to `profile-onboarding` first
  - Then route to `manage-catalog` to create products, features, and offerings

### Scenario B — Brownfield (existing code/SDK, incomplete catalog)

**Conditions:** Some products exist BUT missing offerings or feature assignments.

- Show what exists: list the products and offerings found during state detection.
- Identify what's missing:
  - Products exist but no offerings → route to `manage-catalog` to create offerings and assign features.
  - Products and offerings exist but SDK not integrated → route to `implement-licensing`.
  - Suggest `analyze-codebase` if the user wants to discover additional licensable capabilities.

### Scenario C — Returning User (catalog exists, previous progress)

**Conditions:** Products AND offerings exist.

- Show progress summary: "You have X products, Y offerings configured."
- Skip completed steps and determine the next logical action:
  - If no SDK integration done → route to `implement-licensing`
  - If SDK integrated but no feature gating → route to `implement-licensing` (which routes to `implement-feature`)
  - If everything is set up → route to `analyze-codebase` for optimization, `design-monetization` for strategy refinement, or `scenario-advisor` for licensing model evaluation

## Step 4: Quick Start Mode

**Available only for Greenfield scenario (Scenario A).**

Collapse the full onboarding journey to 2 user interactions with smart defaults:

**Interaction 1: "What features does your app have?"**
- Ask the user to describe their app's capabilities in plain language.
- Map the user's description to features using the glossary:
  - Binary capabilities → ProductAccessFeature (e.g., "premium dashboard" → FeatureKey: `premium-dashboard`)
  - Metered capabilities → ProductRateLimitFeature (e.g., "API calls" → FeatureKey: `api-calls`)

**Interaction 2: "How do you want to charge?"**
- Present options:
  - **Free trial + paid subscription** — 14-day Trial offering (all features Allowed) + Monthly Subscription offering
  - **One-time purchase** — Perpetual offering (all features Allowed)
  - **Usage-based** — Subscription offering with RateLimit features metered
- Map the user's choice to an offering structure with smart defaults.

After both interactions, route to `manage-catalog` with the pre-filled context (product name, features, offering structure) so the catalog can be created with minimal additional input.

## Step 5: Intent Routing (Fallback)

If none of the above scenarios apply cleanly, or if the user expresses a specific intent that doesn't match the detected scenario, present the full routing table with state-aware annotations:

| User Intent | Skill | Description | Status |
|------------|-------|-------------|--------|
| "Set up my product catalog" | `manage-catalog` | Product, feature, offering lifecycle | _(annotate if products/offerings already exist)_ |
| "Integrate licensing into my app" | `implement-licensing` | End-to-end SDK integration | _(annotate if SDK setup detected)_ |
| "Analyze my codebase for licensing opportunities" | `analyze-codebase` | Project analysis and capability discovery | |
| "Design my pricing strategy" | `design-monetization` | Pricing tier and monetization design | |
| "Evaluate licensing scenarios" | `scenario-advisor` | Licensing model recommendation | |
| "Understand the domain model" | `domain-reference` | Entity relationships and concepts | |
| "View my profile or credentials" | `profile-onboarding` | Profile, credentials, and terms review | |

Annotate each row with the user's current progress where applicable (e.g., "✓ 2 products created" or "— not started"). If the user's intent is unclear, summarize the available paths and ask which direction they'd like to go.

</process>

<success_criteria>
- Session is established via `register_or_login` with new/returning user detection
- State is detected by querying `getting_started`, `profile`, `product`, and `offering` tools
- User is classified into the correct scenario (Greenfield, Brownfield, or Returning)
- User is routed to the correct next action based on their progress and intent
- Quick Start mode is offered to greenfield users and collapses to 2 interactions
</success_criteria>
