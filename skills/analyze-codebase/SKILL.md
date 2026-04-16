---
name: analyze-codebase
description: Codebase analysis with context flow — scans your project to identify features worth licensing and classifies each as a feature flag or usage limit
auto-invoke:
  - "User wants to identify what capabilities in their app could be licensed"
  - "User asks what parts of their codebase are licensable"
  - "User wants help figuring out what to monetize in their application"
tags: [discovery, analysis, codebase, capabilities]
category: discovery
allowed-tools: []
argument-hint: ""
tier: 2
invoked-by: [getting-started]
disable-model-invocation: true
---

<input-context>
Receives from getting-started:
- userScenario: "greenfield" | "brownfield" | "returning" — informs scanning depth
- detectedState (optional): { hasProducts } — if products exist, compare against existing catalog

If invoked without upstream context, performs full project scan.
</input-context>

<output-context>
Provides to scenario-advisor:
- capabilities: [{ name, type: "access" | "ratelimit", sourceFile, reason, suggestedFeatureKey }]
- techStack: detected technology stack (e.g., "React + Express", ".NET 8 Web API")

Discovery chain: analyze-codebase [capabilities] → scenario-advisor [selectedScenario] → design-monetization [pricingPlan] → manage-catalog
</output-context>

<state-detection>
Before scanning:
1. Check if previous analysis exists in conversation context
2. If `detectedState.hasProducts`, call `product_feature` (list) to compare existing features against codebase findings

Based on state:
- No prior analysis → Full scan (Step 1-3)
- Prior analysis exists → Show previous results, ask if user wants to rescan
- Existing catalog features → Highlight gaps (capabilities not yet in catalog)
</state-detection>

<objective>
Analyze a user's project to identify capabilities worth licensing. Read project configuration files and key source files (controllers, services, middleware) to produce a structured table of identified capabilities with suggested feature key (the unique identifier for this capability), feature type — either a feature flag (called an Access feature in Monaiq) or a usage limit (called a RateLimit feature in Monaiq) — and reasoning.
</objective>

<process>

## Prerequisites

- Fetch `monaiq://patterns/scenarios` for feature type taxonomy and classification guidance
- Fetch `monaiq://domain/model` for entity context

## Step 1: Scan Project Structure

Perform a deep scan of the user's project:

- Read project config files: `package.json`, `*.csproj`, `Program.cs`, `Startup.cs`, `appsettings.json`, `tsconfig.json`
- Identify the tech stack (React, .NET, Node, etc.) and project structure
- Determine key directories containing business logic (controllers, services, middleware, routes, API handlers)

## Step 2: Identify Licensable Capabilities

Read key source files — controllers, services, middleware, route handlers — and identify capabilities that could be gated:

- Premium features or modules
- API endpoints with business value
- Data exports or report generation
- Third-party integrations
- Compute-intensive operations
- Admin or management tools

For each identified capability, note the source file, the functionality, and why it's licensable.

## Step 3: Classify Capabilities

Using the feature type taxonomy from `monaiq://patterns/scenarios`, classify each capability:

- **Feature flag** (called `ProductAccessFeature` in Monaiq): Binary yes/no — user either has access or doesn't. Use for premium modules, feature flags, advanced tools, exclusive content.
- **Usage limit** (called `ProductRateLimitFeature` in Monaiq): Metered with a quota per time window. Use for API calls, data exports, batch operations, compute units.

Apply the heuristic: "Is the answer yes/no?" → Feature flag. "Is the answer how many?" → Usage limit.

## Step 4: Present Findings

Present a structured table of identified capabilities:

| Capability | Source File | Suggested FeatureKey | Feature Type | Reasoning |
|-----------|-------------|---------------------|--------------|-----------|
| [capability name] | [file path] | [suggested-key] | Access / RateLimit | [why this type] |

Follow with a brief narrative summary: how many capabilities were found, the split between access-gate and rate-limited, and observations about the application's licensing potential.

## Step 5: Suggest Next Step

After presenting findings, suggest: "To map these capabilities to a licensing scenario, run the `scenario-advisor` skill with these findings."

</process>

<success_criteria>
- Project config files and key source files were read (not just config)
- Each identified capability has a suggested FeatureKey, feature type classification, and reasoning
- Classifications align with `monaiq://patterns/scenarios` feature type taxonomy
- Findings presented as structured table plus narrative summary
- `scenario-advisor` suggested as next step
</success_criteria>
