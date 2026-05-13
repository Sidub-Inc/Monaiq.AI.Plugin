---
name: analyze-codebase
description: "Use when: analyzing an existing codebase to identify licensable capabilities, premium features, API quotas, exports, integrations, or monetization candidates; classifies findings as access gates or rate limits."
agent: monaiq
auto-invoke:
  - "User wants to identify what capabilities in their app could be licensed or monetized"
  - "User asks what parts of their codebase are licensable or worth gating"
  - "User wants help figuring out what to monetize in their application — has existing code to analyze"
tags: [discovery, analysis, codebase, capabilities]
category: discovery
allowed-tools: [Read, Grep, Glob, product_feature, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
tier: 2
invoked-by: [getting-started]
---

<objective>
Analyze an existing app and produce an evidence packet of licensable capabilities, classified as Access or RateLimit features, with suggested FeatureKey values and downstream routing. This skill is read-only except for journal updates.
</objective>

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

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only read-only codebase evidence, capability classification, and downstream discovery handoff. Uses `_shared/workflows/checkpoint.md` for `CHECKPOINT-ANALYSIS-SCOPE`.
</execution_context>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — journal startup applies because catalog gap analysis writes journal evidence).
</monaiq-agent-handoff>

<workflow>
1. Run `_shared/workflows/startup.md` for `analyze-codebase`.
2. **Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **codebase-evidence emission**. Evidence sources, in priority order: detected project files (csproj/package.json/manifests), business-code areas matching scenario taxonomy, existing `product_feature` list when catalog exists, prior journal analyses, `monaiq://patterns/scenarios`, `monaiq://platforms/api-surface/{platform}`. The recommendation is the classified capability list with FeatureKey suggestions, not a question about which capabilities to include.
3. Resolve `monaiq://patterns/scenarios`, `monaiq://domain/model`, and platform resources such as `monaiq://platforms/api-surface/{platform}` or `monaiq://platforms/pitfalls/{platform}` when the stack is known before classifying capabilities. Stop before recommendation if taxonomy or domain context is unavailable.
3. Determine scan scope from route context, prior journal state, existing conversation analysis, and project structure. Use `CHECKPOINT-ANALYSIS-SCOPE` before broad scans or scope changes.
4. If products exist, call `product_feature` list and compare catalog features against codebase findings so the output can distinguish existing coverage from gaps.
5. Scan project configuration and key business-code areas. Keep evidence bounded: file paths, area summaries, observed patterns, and confidence; do not quote secrets or dump full files.
6. Classify findings as Access or RateLimit features using fetched scenario/domain taxonomy. Generate stable suggested FeatureKey values.
7. Present the evidence packet using `_shared/response-patterns.md` "Evidence Backing" plus source areas.
8. If missing or stale evidence prevents a confident recommendation, route to `analyze-codebase`, profile/catalog state detection, or the narrowest prerequisite journey step before catalog/pricing recommendations.
9. Output the codebase evidence packet for `scenario-advisor`, coalesce analysis artifacts, checklist progress, and handoff context into the completion workflow, then call `skill_completed` once.
</workflow>

<reference>
## State Routing Reference

- No prior analysis -> full scan.
- Prior analysis exists -> summarize prior findings and rescan only if scope changed or evidence is stale.
- Existing catalog features -> highlight gaps between source capabilities and catalog features.

## Scan Project Structure

Perform a deep scan of the user's project:

- Read project config files: `package.json`, `*.csproj`, `Program.cs`, `Startup.cs`, `appsettings.json`, `tsconfig.json`
- Identify the tech stack (React, .NET, Node, etc.) and project structure
- Determine key directories containing business logic (controllers, services, middleware, routes, API handlers)

## Identify Licensable Capabilities

Read key source files — controllers, services, middleware, route handlers — and identify capabilities that could be gated:

- Premium features or modules
- API endpoints with business value
- Data exports or report generation
- Third-party integrations
- Compute-intensive operations
- Admin or management tools

For each identified capability, note the source file, the functionality, and why it's licensable.

## Classify Capabilities

Using the feature type taxonomy from `monaiq://patterns/scenarios`, classify each capability:

- **Feature flag** (called `ProductAccessFeature` in Monaiq): Binary yes/no — user either has access or doesn't. Use for premium modules, feature flags, advanced tools, exclusive content.
- **Usage limit** (called `ProductRateLimitFeature` in Monaiq): Metered with a quota per time window. Use for API calls, data exports, batch operations, compute units.

Apply the heuristic: "Is the answer yes/no?" → Feature flag. "Is the answer how many?" → Usage limit.

## Present Findings

Present a structured table of identified capabilities:

| Capability | Source File | Suggested FeatureKey | Feature Type | Reasoning |
|-----------|-------------|---------------------|--------------|-----------|
| [capability name] | [file path] | [suggested-key] | Access / RateLimit | [why this type] |

Follow with a brief narrative summary: how many capabilities were found, the split between access-gate and rate-limited, and observations about the application's licensing potential.

## Suggest Next Step

After presenting findings, suggest: "To map these capabilities to a licensing scenario, run the `scenario-advisor` skill with these findings."

</reference>

<success_criteria>
- Project config files and key source files were read (not just config)
- Each identified capability has a suggested FeatureKey, feature type classification, and reasoning
- Classifications align with `monaiq://patterns/scenarios` feature type taxonomy
- Findings distinguish existing catalog coverage from codebase gaps when product features are available
- Findings presented as structured table plus narrative summary
- `scenario-advisor` suggested as next step
</success_criteria>
