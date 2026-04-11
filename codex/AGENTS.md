# Monaiq Agent Instructions

Monaiq is a licensing and monetization platform built on a single-agent, tiered-skill architecture. The monaiq agent uses MCP tools, resources, and skills to guide users from discovery through implementation. This document provides the shared domain knowledge and architectural reference for working with Monaiq.

## Domain Glossary

| Entity | Description | Key Fields |
|--------|-------------|------------|
| **Customer** | Reseller account that owns products and issues licenses | `ClientId` (Guid), `ProfileStatus`, `ResellerStatus` |
| **Product** | Catalog item representing a software product | `Code`, `Name`, `Status` (Draft/Active/Deprecated) |
| **ProductFeature** | Polymorphic feature definition — Access (binary gate) or RateLimit (metered) | `FeatureKey` (string — critical linking field) |
| **ProductOffering** | Pricing unit bundling features with billing configuration | `LicenseClassification` (Trial/Subscription/Perpetual), `BaseRate`, `Interval` |
| **FeatureOffering** | Polymorphic value assignment — `ServiceAccessLevel` (Denied/Allowed) or `SampleSeconds` + `RateLimit` | `FeatureKey` must match a ProductFeature |
| **License** | Issued entitlement created from an offering | `LicenseClassification`, `IssueDate`, `ExpiryDate`, `Signature` |
| **LicenseFeature** | Runtime entitlement for a specific feature within a license | `FeatureKey` — matches across catalog and runtime |
| **EncodedCredential** | Opaque string containing license ID + cryptographic keys for SDK configuration | Used in `LicensingServiceOptions.EncodedCredential` |
| **FeatureKey** | The critical linking field that connects ProductFeature → FeatureOffering → LicenseFeature | String, unique per product |

## Tool Routing

| Category | Tools | When to Use |
|----------|-------|-------------|
| **session** | `register_or_login` | Authentication and session establishment — required before all other operations |
| **onboarding** | `getting_started`, `profile` | First-time setup, onboarding checklist, credential retrieval, terms review |
| **catalog** | `product`, `product_feature`, `offering`, `feature_offering` | Product catalog management — CRUD operations for the product-to-offering lifecycle |
| **integration** | `implement_base`, `implement_feature`, `implement_purchase_flow` | SDK integration guidance — step-by-step workflows for .NET and React |

## MCP Resources

| Resource URI | Content | When to Use |
|-------------|---------|-------------|
| `monaiq://domain/model` | Entity relationships, field definitions, polymorphic hierarchies, invariants | Understanding domain concepts, answering entity relationship questions |
| `monaiq://domain/namespaces` | Authoritative type-to-namespace mappings for .NET and React | Code generation, resolving import/using statements |
| `monaiq://sdk/{language}/setup` | Per-language SDK setup guides (language: `dotnet`, `node`, `react`) | SDK integration, package installation, configuration |
| `monaiq://patterns/scenarios` | Composable building blocks for licensing scenarios: feature types, offering models, and enforcement patterns with Monaiq entity mappings | Recommending licensing models, classifying features by application type |
| `monaiq://patterns/pricing` | Composable pricing patterns showing how industry pricing models map to Monaiq billing primitives (one-time, subscription, consumption) | Designing pricing tiers, choosing billing models |
| `monaiq://troubleshooting` | Structured diagnostic decision trees for setup, authentication, validation, and consumption error categories | Diagnosing SDK integration issues |

## Tiered Skill Architecture

Monaiq skills follow a 3-tier progressive disclosure model. Users interact with Tier 1 entry points, which route to deeper skills as the journey progresses. Only Tier 1 skills are auto-invocable — Tier 2 and Tier 3 skills are invoked by upstream skills based on user intent.

- **Tier 1 — Entry Points:** Auto-invocable skills that handle initial user contact and route to appropriate workflows.
- **Tier 2 — Mid-Journey:** Invoked by Tier 1 entry points for core catalog and integration workflows.
- **Tier 3 — Deep Specialization:** Invoked by upstream skills for targeted tasks like feature implementation, pricing design, or domain questions.

| Skill | Tier | Category | Auto-Invoke | Invoked By |
|-------|------|----------|-------------|------------|
| `getting-started` | 1 | onboarding | Yes | user |
| `troubleshoot-integration` | 1 | integration | Yes | user |
| `manage-catalog` | 2 | catalog | No | getting-started |
| `implement-licensing` | 2 | integration | No | getting-started |
| `analyze-codebase` | 2 | discovery | No | getting-started |
| `design-monetization` | 3 | strategy | No | analyze-codebase, getting-started |
| `scenario-advisor` | 3 | discovery | No | analyze-codebase, getting-started |
| `domain-reference` | 3 | domain | No | getting-started, manage-catalog, implement-licensing |
| `profile-onboarding` | 3 | onboarding | No | getting-started |
| `implement-feature` | 3 | integration | No | implement-licensing, manage-catalog |
| `implement-purchase-flow` | 3 | integration | No | implement-licensing, manage-catalog |

See `Skills/ROUTING-MAP.md` for the full intent-to-skill routing topology and mermaid diagram.

## Skill Data-Flow Map

Skills pass structured context objects downstream to avoid redundant state detection. Each skill declares its input and output context in its frontmatter.

### Context Flow Chain

```
getting-started [userScenario, detectedState, quickStartSpec]
  → manage-catalog [catalogSpec, catalogComplete]
    → implement-licensing [sdkConfig, targetFeatures]
      → implement-feature [featureImpl]
      → implement-purchase-flow [checkoutImpl]
```

### Context Objects

| Context Object | Source Skill | Target Skill(s) | Key Fields |
|---------------|-------------|-----------------|------------|
| `userScenario` | getting-started | manage-catalog, implement-licensing, analyze-codebase | `"greenfield"` \| `"brownfield"` \| `"returning"` |
| `detectedState` | getting-started | manage-catalog, implement-licensing, analyze-codebase | `hasProducts`, `hasOfferings`, `profileComplete`, `resellerEnabled` |
| `quickStartSpec` | getting-started | manage-catalog | `appDescription`, `pricingChoice` (greenfield only) |
| `catalogSpec` | manage-catalog | implement-licensing, implement-feature, implement-purchase-flow | `productCode`, `features[]`, `offerings[]`, `assignments[]` |
| `catalogComplete` | manage-catalog | implement-licensing | `true` when product has features, offerings, and assignments verified |
| `sdkConfig` | implement-licensing | implement-feature, implement-purchase-flow | `language`, `credentialSource`, `serviceOptionsConfigured`, `diRegistered` |
| `targetFeatures` | implement-licensing | implement-feature | `featureKey`, `featureType` per feature |
| `featureImpl` | implement-feature | (terminal) | `implementedFeatures[]`, `buildVerified` |
| `checkoutImpl` | implement-purchase-flow | (terminal) | `architecture`, `offeringCodes[]`, `checkoutRoute`, `successHandler` |

If a skill is invoked directly without upstream context, it runs its own state detection to build equivalent context.

## Journey Progress

Users can check their current position in the Monaiq journey by asking: **"What's my current status?"** or **"Check my progress"** or **"Where am I in the journey?"**

The agent responds by checking session state and calling `product` (list), `offering` (list), and `profile` to determine the current state:

| State | Indicator | Next Recommended Action |
|-------|-----------|------------------------|
| **Not started** | No session established | Call `register_or_login` to authenticate |
| **Authenticated** | Session active, no products | Start with `getting-started` skill for onboarding |
| **Catalog started** | Products exist, no offerings or incomplete assignments | Continue with `manage-catalog` to add features and offerings |
| **Catalog complete** | Product with features, offerings, and assignments verified | Proceed to `implement-licensing` for SDK integration |
| **SDK integrated** | SDK packages installed, DI registered, credentials configured | Use `implement-feature` to add feature checks |
| **Features implemented** | Feature checks in codebase, build verified | Use `implement-purchase-flow` for checkout or explore `design-monetization` for pricing strategy |

## Agent

Monaiq uses a single agent with two behavioral modes that adapt automatically based on user intent.

| Agent | Modes | Tool Access | File |
|-------|-------|-------------|------|
| **monaiq** | Discovery, Implementation | Full access | `Agents/monaiq.md` |

### Behavioral Modes

**Discovery Mode** — Active when users analyze codebases, evaluate licensing scenarios, design pricing strategies, or ask domain questions. Favors read-only tool operations. A confirmation gate fires before any create/update/delete operation.

**Implementation Mode** — Active when users create products, configure offerings, integrate SDKs, or manage their catalog. Full tool access for create/update/delete operations.

Mode switching is automatic based on user intent signals — no manual agent selection required.

## Behavioral Constraints

- **Use exact tool names** as listed in the Tool Routing table — do not abbreviate or alias
- **Reference MCP resources** using the `monaiq://` URI scheme
- **FeatureKey consistency** — when creating features and assigning them to offerings, the `FeatureKey` string must match exactly across `product_feature` and `feature_offering` operations
- **Polymorphic type matching** — Access features use `ServiceAccessFeatureOffering` assignments; RateLimit features use `RateLimitFeatureOffering` assignments. Mixing types causes runtime failures
- **Credential security** — never persist `ApiKey` to disk or expose it in frontend code. Use environment variables or secrets managers for production
- **Session establishment** — `register_or_login` should be called before other tools. If a tool returns an authentication error, call `register_or_login` first and retry
