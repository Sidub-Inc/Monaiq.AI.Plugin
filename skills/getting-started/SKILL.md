---
name: getting-started
description: "Use when: onboarding a new or returning Monaiq user, establishing a session, checking profile/catalog state, choosing the next licensing workflow, or asking where to start."
agent: monaiq
auto-invoke:
  - "User wants to get started with Monaiq licensing and has not yet authenticated or set up products"
  - "User asks what Monaiq can do for them and needs an overview, not a specific task"
  - "User is new to Monaiq and needs orientation — no existing products or SDK integration"
  - "User asks 'where do I start' or 'how do I begin' with licensing"
tags: [onboarding, getting-started, orchestration, state-detection]
category: onboarding
allowed-tools: [register_or_login, getting_started, profile, product, offering, monaiq_journal, fetch_step_resources, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__getting_started, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__offering, mcp__plugin_monaiq_monaiq__monaiq_journal, mcp__plugin_monaiq_monaiq__fetch_step_resources]
tier: 1
invoked-by: [user]
---

<objective>
State-aware onboarding that detects user progress, classifies their scenario (greenfield, brownfield, or returning), and routes intelligently to the next action. Supports Quick Start mode for greenfield users to collapse the full journey into 2 interactions.
</objective>

<execution_context>
Executable Monaiq workflow protocol. Direct skill invocation is first-class; the custom monaiq agent is optional convenience only. This skill is authoritative when invoked directly, provided it follows the required reading, journal startup, returned file-operation application, checkpoint, prerequisite-stop, and success criteria below before any consequential work.
</execution_context>

<required_reading>
- `ROUTING-MAP.md` for the shared route/checkpoint contract, route packet vocabulary, specialist phase boundaries, and direct invocation fallback.
- `maintain-implementation-journal.md` for journal file operation application, checkpoint anatomy, and approval-result recording.
- `_shared/workflows/startup.md` for readiness checks, journal startup, resume behavior, and user-visible blocker reports.
- `_shared/response-patterns.md` and `_shared/handoff-schemas.md` for status summaries and route packet persistence.
- `_shared/protocols.md` for read-only versus mutation tool-operation rules.
- `monaiq://protocols/implementation-journal` through `fetch_step_resources` before journal startup or resume decisions.
</required_reading>

<direct-invocation-contract>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant). For this skill, that means: call `monaiq_journal get_state` unless a fresh state packet is already available in this turn, call `monaiq_journal init` when state is absent, apply returned `.monaiq/*` file operations after path validation, call `skill_started` for a new or stale journey, present and save `CHECKPOINT-WORKFLOW-START`, then route to the narrowest specialist. Stop before consequential work when journal, route, catalog, SDK, code evidence, or MCP readiness prerequisites are missing.
</direct-invocation-contract>

<process_steps>
1. Call `monaiq_journal get_state`; if no state exists, call `monaiq_journal init` and apply all returned `.monaiq/*` file operations using the **File Operation Application Protocol** in `_shared/protocols.md` before continuing.
   - **If `resumeAvailable == true`:** render a **Status Summary** using `_shared/response-patterns.md` (current gate, last checkpoint, open blockers/questions, backend summary: profile, products, offerings, SDK, and one primary next skill recommendation). Then invoke the **Host-Native Ask Pattern** (see `_shared/protocols.md` § Host-Native Ask Pattern) with these three options: (1) "Continue from [last checkpoint]", (2) "Start fresh", (3) "Show journal". **Block until the user responds before any other step.**
     - **Continue** → skip to step 7 (CHECKPOINT-WORKFLOW-START was already recorded in the prior session; do not re-present it)
     - **Start fresh** → proceed to step 2 as a new-user intake (all steps including step 6 apply)
     - **Show journal** → display `.monaiq/JOURNAL.md` content, then re-present step 1's widget
   - **If `resumeAvailable == false`:** proceed to step 2.
2. Fetch `monaiq://protocols/implementation-journal` and read the shared route/checkpoint contract plus `_shared/workflows/startup.md`.
3. Verify `.monaiq/STATE.md`, `.monaiq/JOURNAL.md`, and `.monaiq/CHECKPOINTS` exist when the returned operations indicate they should. Verify the master journey checklist is present in `.monaiq/STATE.md`; if it is absent, use `monaiq_journal init` or `update_checklist_progress` from `monaiq://protocols/implementation-journal` to restore the compact resume/control packet before routing.
4. Call `monaiq_journal skill_started` for `getting-started` only for a new, stale, or materially changed journey; skip it when resuming from a fresh state packet.
5. Inspect profile, catalog, code, journal evidence, and master journey checklist gates before asking detailed questions.
6. Present `CHECKPOINT-WORKFLOW-START` with scenario, target app/platform, `activePlatform`, `targetProject`, `outOfScopePlatforms`, intended outcome, and any inferred assumptions; continue only after recording the approval result. **Omit this step when a returning user chose "Continue" in step 1 — their prior session already recorded this checkpoint.**
7. Inspect profile, catalog, code, journal evidence, and master journey checklist gates before asking detailed questions. (Returning users who chose "Continue" resume here directly after step 1.)
8. Emit one route packet with the exact shared fields, including `activePlatform`, `targetProject`, and `outOfScopePlatforms`, and hand off to the narrowest missing prerequisite or specialist skill. Emit the **Next Step Signal** as defined in `_shared/workflows/completion.md` step 9:
   - When routing to catalog creation: `**Next:** Invoke \`manage-catalog\` — create your product catalog so the SDK has offerings to enforce.`
   - When routing to SDK integration: `**Next:** Invoke \`implement-licensing\` — integrate the Monaiq SDK into your application to enforce licensing.`
</process_steps>

<resume-mode>
<!-- Note: Resume Mode is now an executable pre-flight gate in <process_steps> step 1. This section provides supplementary guidance for the Resume route. -->
When `.monaiq/STATE.md` exists, enter Resume Mode before new intake. Render a Status Summary using `_shared/response-patterns.md`: current gate, last checkpoint, open blockers/questions, backend summary (profile, products, offerings, SDK), and one primary next skill recommendation. Only present 1-2 alternatives when the route is genuinely ambiguous. Do not re-ask questions already answered by the route packet, journal evidence, or backend facts.
</resume-mode>

<readiness-blocker-report>
If `fetch_step_resources`, `monaiq_journal`, or a required MCP tool is unavailable, render a user-visible READINESS BLOCKER report with: missing capability, affected skill/gate, what can continue, what is blocked, recovery action, and no-secret diagnostic summary. Continue only with read-only recommendations or domain explanation; block catalog mutation, credential/config writes, application behavior edits, checkpoint recording, and validation remediation until readiness is restored.
</readiness-blocker-report>

<anti_patterns>
- Do not treat direct skill invocation as a lower-quality path.
- Do not skip required journal startup, `monaiq_journal init`, returned operation application, `skill_started` for new or stale journeys, or hard checkpoints because the custom agent is unavailable.
- This intake/router skill must not perform substantive catalog, SDK, feature, purchase, or troubleshooting work itself.
- Do not ask low-value questions when existing code, journal, profile, or catalog evidence supports an inference that can be confirmed in the next checkpoint.
</anti_patterns>

<success_criteria>
- Direct invocation initializes or resumes `.monaiq` state, applies returned `.monaiq/*` file operations, records `skill_started` when the journey is new or stale, and enforces `CHECKPOINT-WORKFLOW-START`.
- The route packet preserves `scenario`, `targetApp`, `platform`, `activePlatform`, `targetProject`, `outOfScopePlatforms`, `profileState`, `catalogState`, `codeEvidenceSummary`, `journalEvidenceSummary`, `assumptions`, `recommendedSkill`, `recommendationRationale`, `checkpointName`, and `authorizedBy`.
- The user sees one business-readable recommendation with compact technical backing and a specialist handoff.
- Missing prerequisites route to the narrowest missing prerequisite and stop before consequential work.
</success_criteria>

<glossary-for-humans>

When users describe what they want using everyday language, map their terms to Monaiq domain concepts:

| User Says | Monaiq Concept | Details |
|-----------|---------------|---------|
| "subscription" / "recurring billing" | ProductOffering | LicenseClassification = Subscription |
| "free trial" / "trial period" | ProductOffering | LicenseClassification = Trial |
| "one-time purchase" / "perpetual license" | ProductOffering | LicenseClassification = Perpetual |
| "license key" / "activation key" | EncodedCredential | Returned after purchasing an offering, then stored by the app |
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
- route packet: { scenario, targetApp, platform, activePlatform, targetProject, outOfScopePlatforms, profileState, catalogState, codeEvidenceSummary, journalEvidenceSummary, assumptions, recommendedSkill, recommendationRationale, checkpointName, authorizedBy }

Routes to:
- manage-catalog: receives { userScenario, detectedState, quickStartSpec? }
- implement-licensing: receives { userScenario, detectedState }
- analyze-codebase: receives { userScenario, detectedState }
</output-context>

<shared-contract>
Use the shared route/checkpoint contract from `ROUTING-MAP.md` and `maintain-implementation-journal.md`. `ROUTING-MAP.md` supplies the route packet vocabulary, phase boundaries, and specialist handoff semantics; `maintain-implementation-journal.md` supplies checkpoint anatomy and approval-result recording.

Present route recommendations in business-readable language first: what the user is trying to accomplish, why the next skill is the right step, and what it changes for their business journey. Then include compact technical backing for agents: route packet fields, evidence summaries, missing assumptions, checkpoint name, and authorizedBy.
</shared-contract>

<reference>
## Authentication & User Detection

Call the `register_or_login` tool to authenticate and establish a session.

**If the tool indicates no existing account — this is a NEW user:**
- Guide through registration: "Provide your email to create a Monaiq account."
- After registration completes, proceed to Step 2.

**If login succeeds — this is a RETURNING user:**
- Proceed to Step 2 with returning-user context.

## State Detection

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

## Scenario Classification

Based on the detected state, classify the user into one of three scenarios:

### Scenario A — Greenfield (new user, no catalog)

**Conditions:** No products AND no offerings (OR new registration from Step 1).

Present two options:
- **Quick Start** (recommended for new users) — collapses the journey to 2 interactions. See Step 4.
- **Full Guided Flow** — step-by-step setup:
  - If profile is incomplete → route to `profile-onboarding` first
  - Then route to `manage-catalog` to create products, features, and offerings

**Next:** Invoke `manage-catalog` — create your product catalog so the SDK has offerings to enforce.

### Scenario B — Brownfield (existing code/SDK, incomplete catalog)

**Conditions:** Some products exist BUT missing offerings or feature assignments.

- Show what exists: list the products and offerings found during state detection.
- Identify what's missing:
  - Products exist but no offerings → route to `manage-catalog` to create offerings and assign features.
  - Products and offerings exist but SDK not integrated → route to `implement-licensing`.
    **Next:** Invoke `implement-licensing` — integrate the Monaiq SDK into your application to enforce licensing.
  - Suggest `analyze-codebase` if the user wants to discover additional licensable capabilities.
  - For projects that already have a user/customer/tenant table, route to the `implement-licensing` Brownfield Credential Contract so ownership scope is decided before credential-persistence guidance.

### Scenario C — Returning User (catalog exists, previous progress)

**Conditions:** Products AND offerings exist.

- Show progress summary: "You have X products, Y offerings configured."
- Skip completed steps and determine the next logical action:
  - If no SDK integration done → route to `implement-licensing`
  - If SDK integrated but no feature gating → route to `implement-licensing` (which routes to `implement-feature`)
  - If everything is set up → route to `analyze-codebase` for optimization, `design-monetization` for strategy refinement, or `scenario-advisor` for licensing model evaluation

## Quick Start Mode

**Available only for Greenfield scenario (Scenario A).** Collapse onboarding to two user interactions with strong defaults.

**Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **onboarding → catalog**. Evidence sources, in priority order: route-packet handoff from broad intake, `analyze-codebase` results (invoke when reachable codebase + no prior analysis), journal decisions, conversation cues, `monaiq://patterns/scenarios`. Recommendation precedes evidence in every reply.

**Interaction 1 — starter feature set.** Recommend a starter feature set inferred from `analyze-codebase` (or the prior analysis on the route packet/journal). The recommendation is the default; the host-native question lets the user approve, refine features, refine pricing, or pause to inspect the analysis. Map binary capabilities to `ProductAccessFeature` and metered capabilities to `ProductRateLimitFeature`. Confirm via `CHECKPOINT-PRE-CATALOG-MUTATION` when the user advances to catalog creation.

**Interaction 2 — starter pricing model.** Recommend one pricing model (Free + Trial + Subscription | Perpetual | Usage-based) tied to the inferred capability mix and app type. The recommendation is the default; the host-native question lets the user approve, swap models, refine tiers, or hand off to `design-monetization` for deeper pricing work. Confirm via `CHECKPOINT-PRICING-APPROVAL` when handing off, or `CHECKPOINT-PRE-CATALOG-MUTATION` when applying inline.

After both interactions, hand off to `manage-catalog` with the pre-filled `quickStartSpec` context. Quick Start does not create or update catalog entities directly; route to `manage-catalog` before any `product`, `product_feature`, `offering`, or `feature_offering` mutation.

## Intent Routing Fallback

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

Use the Intent Dispatch Table in `ROUTING-MAP.md` before showing this fallback. The dispatch step matches intent, confirms only genuinely ambiguous routes with one short question, then hands off rather than doing specialist work inline.

</reference>

<success_criteria>
- Session is established via `register_or_login` with new/returning user detection
- State is detected by querying `getting_started`, `profile`, `product`, and `offering` tools
- User is classified into the correct scenario (Greenfield, Brownfield, or Returning)
- User is routed to the correct next action based on their progress and intent
- Quick Start mode is offered to greenfield users and collapses to 2 interactions
</success_criteria>
