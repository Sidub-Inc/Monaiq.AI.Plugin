---
name: scenario-advisor
description: "Use when: recommending a licensing model for an app type, comparing subscription, trial, perpetual, freemium, tiered, or usage-based scenarios before pricing design."
agent: monaiq
auto-invoke:
  - "User wants to know which licensing scenario fits their application"
  - "User asks what licensing model to use for their product"
  - "User wants licensing recommendations for their app type"
tags: [discovery, scenarios, recommendations, strategy]
category: discovery
allowed-tools: [Read, Grep, Glob, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "appType (saas|desktop|plugin|package|cli|api)"
tier: 3
invoked-by: [analyze-codebase, getting-started]
---

<objective>
Recommend licensing scenarios for the user's app type using capability evidence, route context, and Monaiq scenario patterns. The skill narrows strategic direction before pricing design; it does not design exact tier prices or mutate catalog data.
</objective>

<input-context>
Receives from analyze-codebase:
- capabilities (optional): [{ name, type, suggestedFeatureKey }] — identified licensable features
- techStack (optional): detected technology stack — used to infer app type

If invoked without upstream context, asks the user about their application type directly.
</input-context>

<output-context>
Provides to design-monetization:
- selectedScenario: { name, model, featureTypes: ["access" | "ratelimit"], offeringModel, enforcement }
- appType: "saas" | "desktop" | "plugin" | "package" | "cli" | "api"

Discovery chain: analyze-codebase [capabilities] → scenario-advisor [selectedScenario] → design-monetization [pricingPlan]
</output-context>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only licensing-scenario recommendation and handoff context; exact prices and catalog mutation belong to `design-monetization` and `manage-catalog`.
</execution_context>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — `CHECKPOINT-SCENARIO-CHOICE` writes a scenario decision).
</monaiq-agent-handoff>

<workflow>
1. Run `_shared/workflows/startup.md` for `scenario-advisor`.
2. Resolve `monaiq://patterns/scenarios`, `monaiq://domain/model`, and platform resources such as `monaiq://platforms/api-surface/{platform}` or `monaiq://platforms/pitfalls/{platform}` when the stack is known before presenting options. Stop before recommendation if scenario building blocks or domain context are unavailable.
3. **Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **scenario choice**. Evidence sources, in priority order: `analyze-codebase` capabilities + tech stack, route packet `appType`, prior journal decisions, `monaiq://patterns/scenarios`, conversation cues. If a codebase is reachable but no analysis evidence exists, route to `analyze-codebase` first.
4. Present 2–3 scenario options with trade-offs and exactly one marked as the recommendation tied to the evidence cite. The host-native question lets the user approve, swap to an alternative, or pause to inspect `monaiq://patterns/scenarios`.
5. Use `CHECKPOINT-SCENARIO-CHOICE` before locking the recommended scenario for pricing design.
6. Output `selectedScenario` and `appType` for `design-monetization`, coalesce scenario choice + checklist progress + handoff context into the completion workflow, then call `skill_completed` once.
</workflow>

<reference>
## State Routing Reference

- Capabilities provided -> infer app type and present scenario options.
- No upstream context -> ask the narrowest app-type question needed.

## Determine Application Type

Infer the application type from available context:

- **SaaS web application** — React/Angular/Next.js frontend, API controllers, multi-tenant patterns
- **Desktop application** — WPF/WinForms/Electron, local file I/O, installer references
- **Plugin/extension** — Plugin manifest, extension API, host app integration
- **Package/library** — NuGet/npm publish config, public API surface, no entry point
- **CLI tool** — Console app, command parsers, argument handling
- **API service** — REST/GraphQL endpoints, auth middleware, rate limiting

If `analyze-codebase` was run previously, use its output (tech stack, identified capabilities) to infer the type. If a codebase is reachable but no analysis evidence exists, route to `analyze-codebase` first rather than asking the user to pick. Only when no codebase is reachable AND no upstream evidence is available, present the application-type list — framed explicitly as an evidence gap — and ask the user to confirm.

## Present Licensing Model Options

Using the composition examples from `monaiq://patterns/scenarios`, present 2–3 licensing model options tailored to the inferred application type:

| Aspect | Option A: [Name] | Option B: [Name] | Option C: [Name] |
|--------|------------------|------------------|------------------|
| **Model** | [e.g., Freemium + Rate Limits] | [e.g., Tiered Subscriptions] | [e.g., Usage-Based] |
| **Feature Types** | [Feature flags + Usage limits] | [Feature flags only] | [Usage limits only] |
| **Pricing Tiers** | [Tier] | [Tier + Add-on] | [Metered] |
| **How Licenses Are Checked** | [Access Gate + Rate Limit] | [Feature Flag] | [Rate Limit] |
| **Best For** | [trade-off summary] | [trade-off summary] | [trade-off summary] |

Reference specific building blocks from `monaiq://patterns/scenarios` by name — do not duplicate resource content.

## Capture User Selection

Present the options with one clearly marked as the recommendation, tied to the evidence summary (codebase capabilities, tech stack, app archetype, prior journal decisions). Use `CHECKPOINT-SCENARIO-CHOICE` to confirm — this is a confirmation prompt, not an open menu. Record the chosen scenario, including any deferral ("you decide") under which the recommendation was applied, in the journal.

## Route to Design

After the user selects a licensing model, suggest: "To design your pricing tiers and feature assignments based on this scenario, run the `design-monetization` skill." Include the selected scenario context (application type, chosen model, feature types) for continuity.

</reference>

<success_criteria>
- Application type was inferred from context (not immediately asked)
- 2–3 scenario options presented with trade-offs
- Options reference building blocks from `monaiq://patterns/scenarios`
- User selected a scenario
- `design-monetization` suggested as next step with scenario context
</success_criteria>
