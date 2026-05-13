---
name: implement-licensing
description: "Use when: integrating the Monaiq licensing SDK into a .NET or React app, installing packages, configuring credentials/endpoints, registering services/providers, or verifying runtime license validation."
agent: monaiq
auto-invoke:
  - "User wants to integrate licensing SDK into their application — has a product catalog already set up"
  - "User wants to add license validation to their project and needs package installation and configuration"
  - "User asks how to set up Monaiq licensing in code — not asking about pricing or catalog design"
tags: [sdk, integration, licensing, setup, dotnet, react]
category: integration
allowed-tools: [Read, Write, Edit, Grep, Glob, Bash, register_or_login, profile, product, product_feature, implement_base, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__implement_base, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "platform (dotnet|dotnet/blazor-server|react|react/vite|react/nextjs)"
tier: 2
invoked-by: [getting-started]
---

<objective>
Integrate the Monaiq licensing SDK into a .NET or React app using `implement_base` and canonical resources as the implementation authority. The skill owns platform detection, credential strategy, package/config/DI setup, runtime validation, and downstream handoff to feature gates or purchase flow.
</objective>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — SDK setup writes code/config/credentials behind hard checkpoints).
</monaiq-agent-handoff>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only SDK setup, credential strategy, validation, and handoff decisions.
</execution_context>

<input-output-contract>
Input from `manage-catalog` or direct invocation: optional `catalogSpec`, optional platform, and route/journal evidence. Direct invocation must detect equivalent platform, SDK, profile, catalog, and journal state before edits.

Output: `sdkConfig: { platform, activePlatform, targetProject, outOfScopePlatforms, credentialSource, serviceOptionsConfigured, diRegistered }` and `targetFeatures` for `implement-feature` or `implement-purchase-flow`.
</input-output-contract>

<tool-first-authority>
implement_base is authoritative for SDK setup. Use `implement_base` and `fetch_step_resources` as the normal implementation guidance paths before code/config/business-logic changes.

SDK reverse-engineering is a plugin guidance defect. Do not inspect DLLs. Do not inspect XML documentation. Do not inspect NuGet package caches. Do not inspect generated binaries to discover SDK signatures, namespaces, setup snippets, endpoint values, or runtime behavior.

If Monaiq tool or resource guidance is missing, contradictory, insufficient, contradicts the installed SDK package/version, or produces a compile failure when followed, stop before further code/config/business-logic changes and record a plugin guidance defect with `monaiq_journal record_error` or `monaiq_journal record_validation_failure`. Include the package name/version, validation command, and compile error. Do not inspect package assemblies, DLLs, XML docs, NuGet caches, generated binaries, or local package folders to discover signatures.
</tool-first-authority>

<checkpoint-workflow-directive>
For `CHECKPOINT-CREDENTIAL-SOURCE`, `CHECKPOINT-PRE-BROWNFIELD-MIGRATION`, `CHECKPOINT-PRE-CREDENTIAL-WRITE`, and any other hard checkpoint, follow `_shared/workflows/checkpoint.md` exactly. Use `checkpoint` in `monaiq_journal save_checkpoint` payloads, record the user's `result`, apply returned `.monaiq/*` file operations, and verify the checkpoint file exists before code/config work continues.
</checkpoint-workflow-directive>

<workflow>
1. Run `_shared/workflows/startup.md` for `implement-licensing`.
2. **Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **base SDK setup**. Cite evidence as a short business-readable evidence summary (1–3 lines, named files/packages, no "based on my analysis" framing). Evidence sources, in priority order: route packet `activePlatform` + `targetProject`, package manifest + DI/provider registration in `targetProject`, profile/session state, journal decisions, `monaiq://sdk/{stack}/setup`, `monaiq://platforms/api-surface/{platform}`, `monaiq://config/endpoints`, `monaiq://docs/anti-patterns/{platform}`. Per-stack code shapes live in `monaiq://sdk/{stack}/setup` and the `implement_base` server response — do not embed framework-specific code in this skill source.
3. Read the master journey checklist from `.monaiq/STATE.md`; this skill owns only the base SDK setup gate. Establish prerequisites: active session, `activePlatform`, `targetProject`, `outOfScopePlatforms`, catalog or target feature context when needed, and canonical resources fetched. Stop before edits when resource guidance is missing.
3. Default to one platform and one target project. If the route packet says `activePlatform = dotnet/blazor-server` and `targetProject` points at a Blazor project, do not install npm packages, create React providers, modify Node assets, or add Console samples unless the user approves a secondary target through `CHECKPOINT-FRAMEWORK-CHOICE`. Apply the same rule in reverse for React/Node targets.
4. Detect current SDK state with resource-backed facts: package manifest, licensing configuration, DI/provider registration, runtime usage, and existing credential storage in the `targetProject` only.
5. Use evidence before asking a new question. Infer credential handling, checkout architecture impact, feature path dependencies, and next steps from codebase evidence, route packet context, profile/session state, catalog facts, and journal decisions. You must confirm inferred decisions in the next existing checkpoint, especially `CHECKPOINT-CREDENTIAL-SOURCE`, `CHECKPOINT-PRE-BROWNFIELD-MIGRATION`, or `CHECKPOINT-PRE-CREDENTIAL-WRITE`, with labeled assumptions. If the app has user-login evidence, infer user-managed credential persistence from user-login evidence and confirm that decision in CHECKPOINT-CREDENTIAL-SOURCE or CHECKPOINT-PRE-BROWNFIELD-MIGRATION instead of asking a separate credential-scope question.
6. Present the plan using `_shared/response-patterns.md` "Evidence Backing" plus active target boundaries and validation plan.
7. Call `implement_base` with `startStep=all` when platform, app context, required resources, and credential/config safety posture are known; use numbered steps only when step-by-step review improves safety.
8. Stop at the relevant checkpoint before consequential changes: `CHECKPOINT-FRAMEWORK-CHOICE`, `CHECKPOINT-CREDENTIAL-SOURCE`, `CHECKPOINT-PRE-BROWNFIELD-MIGRATION`, or `CHECKPOINT-PRE-CREDENTIAL-WRITE`. `CHECKPOINT-FRAMEWORK-CHOICE` is mandatory before adding any platform listed in `outOfScopePlatforms`. `CHECKPOINT-PRE-CREDENTIAL-WRITE` is mandatory before ApiKeys, IssuerClientId, endpoint URLs, SDK provider configuration, appsettings, user-secrets, `.env`, or persisted Monaiq configuration; record the user's result before proceeding.
9. Apply package/config/DI/runtime edits using only `implement_base` and fetched resource guidance for the active platform and target project. Do not store raw ApiKeys, EncodedCredential values, `.env` contents, or user-secrets content in `.monaiq`.
10. Build and validate. On failure, call `monaiq_journal record_validation_failure` with command, observed result, likely cause, blocked next action, retry choices, package/version facts, and checkpoint requirement before remediation. If the failure indicates Monaiq canonical guidance is wrong for the installed SDK, stop and report a plugin guidance defect instead of reverse engineering package artifacts.
11. Coalesce changed paths, validation proof, base SDK setup checklist progress, route boundaries, and `sdkConfig` handoff through `_shared/workflows/completion.md`. Call `update_checklist_progress` for base SDK setup only after source skill, relevant MCP tool, canonical resources, and checkpoint/journal evidence prove the gate is complete before marking the checklist gate complete. Save `CHECKPOINT-SKILL-COMPLETE` with `proofOfDone` only when useful, apply returned file operations using the **File Operation Application Protocol** in `_shared/protocols.md`, call `skill_completed` once, and hand off persisted `sdkConfig`.
</workflow>

<reference>
> Each `monaiq://...` URI below must be fetched through `fetch_step_resources` before the related code/config/business-logic edit. The repeated fetch lines exist by design \u2014 they are explicit cues for tool invocation across hosts. The closing "Related Resources" section is the consolidated index.

## State Detection Reference

- Nothing found -> full SDK setup flow.
- Packages installed -> start at credential/configuration.
- Packages and config found -> skip to DI/provider registration.
- Everything configured -> summarize and offer credential recovery or verification workflows.

## Credential Source Reference

Before writing any code, determine how end-users will provide their license credentials.

**Core question:** Do individual users or tenants each provide their own separate license credentials?

| Answer | Credential Source | Typical Use Case |
|--------|------------------|------------------|
| No | Configuration-Based | Single organization license, one credential in config |
| Yes | User-Managed | Multi-tenant SaaS, marketplace apps, per-user subscriptions |

**Configuration-Based** — one license credential stored in application settings covers the entire application. The SDK's built-in configuration-based credential resolver handles this automatically. For the exact configuration shape and resolver type names, resolve `monaiq://platforms/api-surface/{platform}`.

**User-Managed** — each user/tenant provides their own credential at runtime. You must implement a custom credential-provider type that resolves credentials from your storage (user profile, tenant settings, etc.). For the authoritative provider interface / hook signature, resolve `monaiq://platforms/api-surface/{platform}`. Consider the `implement_purchase_flow` tool for embedded in-app purchases that provision credentials automatically.

## Step 2: Install Packages

Install the SDK package for the target stack. For the installation command, package name, and transitive-dependency notes, resolve:

Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

> **Note (.NET):** Requires Sidub.Platform 1.10.38+. Consumers using this version or later do not need any `<ExcludeAssets>` Metalama workaround — the Metalama build tool is no longer a transitive dependency (DEV-08 / RT-6 resolved in Sidub.Platform.Core).

## Step 3: Namespace Reference

For the authoritative type-to-namespace mappings (which imports expose credential types, service interfaces, feature types, and platform core), resolve:

Fetch `monaiq://domain/namespaces` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

Use ONLY the namespaces from the reference. Do not guess or assume type locations.

## Step 4: Configure Service Options

Configure the licensing service with endpoint URIs and (for Configuration-Based credential source) the encoded credential.

- For the platform-specific configuration shape (settings file keys, provider component props, environment variable wiring), resolve:

  Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the endpoint URL values to populate into that configuration, resolve:

  Fetch `monaiq://config/endpoints` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the authoritative property names and their types on the configuration surface, resolve:

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For **User-Managed** credential source, omit the encoded credential from configuration — it is resolved at runtime by your custom credential resolver (see Step 5).

## Step 5: Register Dependency Injection

Register the licensing services with your application's DI / provider graph.

- For the platform-specific registration pattern (service collection extensions in .NET, provider component in React), resolve:

  Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the exact extension-method / component signatures (including the generic overload used for user-managed credential resolvers), resolve:

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For **User-Managed** credential source, register your custom credential-resolver type via the generic registration overload documented in the platform's api-surface resource.

## Step 6: Runtime Usage

Integrate the licensing service into your application's runtime.

- For the platform-specific type signatures — the service interface (.NET) or hook (React) used to request authorization — resolve:

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the runtime wiring narrative — how to inject the service, when to call it, error-state handling — resolve:

  Fetch `monaiq://sdk/{stack}/setup` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

For known platform-specific pitfalls (null-semantics differences, provider remount behavior), resolve:

Fetch `monaiq://platforms/pitfalls/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Step 7: Displaying License State

Your application often needs to render the user's current license, entitlements, or consumption
— entitled features, rate-limit usage, expiry. The Monaiq SDK exposes this snapshot in-process; it
does **not** ship UI chrome. Render with your existing component library.

- For the platform-specific signature (`LicensingClient.getState()` for React,
  `ILicenseStateProvider.GetState()` for .NET) and the `LicenseStateView` shape, resolve:

  Fetch `monaiq://platforms/api-surface/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

- For the canonical post-purchase refresh + snapshot pattern, resolve:

  Fetch `monaiq://platforms/manifest/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

Typical render patterns (implementer's choice, no SDK opinion):
- **Badge** — show "Licensed · {planName}" or a compact status chip in the app header.
- **Settings card** — dedicated panel listing features + consumption bars on an account/settings page.
- **Dashboard widget** — surface near-limit features on the main landing view.

> The SDK ships no UI. Build the view with your existing component library.

## Verification

After completing the integration:

1. Build the project — no compilation errors.
2. Run the application and verify the SDK's authorization call returns a non-null result.
3. Check logs for any licensing configuration exception — this indicates incomplete credential fields.

## Related Tools

- `implement_base` — Interactive step-by-step SDK integration (follows the same 6-area structure; call `startStep=1` through `startStep=6` consecutively).
- `register_or_login` — Establish a session before integration.
- `profile` — Retrieve credentials and onboarding status.

## Related Resources

- `monaiq://sdk/dotnet/setup` — .NET SDK setup narrative.
- `monaiq://sdk/react/setup` — React SDK setup narrative.
- `monaiq://platforms/api-surface/dotnet` — .NET method signatures and null semantics.
- `monaiq://platforms/api-surface/react` — React method signatures and null semantics.
- `monaiq://platforms/manifest/dotnet` — Full machine-readable .NET platform manifest.
- `monaiq://platforms/manifest/react` — Full machine-readable React platform manifest.
- `monaiq://platforms/pitfalls/dotnet` — .NET known issues.
- `monaiq://platforms/pitfalls/react` — React known issues (including provider-remount behavior).
- `monaiq://docs/anti-patterns/dotnet` — .NET licensing anti-patterns to avoid.
- `monaiq://docs/anti-patterns/react` — React licensing anti-patterns to avoid.
- `monaiq://domain/namespaces` — Authoritative namespace-to-type mappings.
- `monaiq://domain/model` — Cross-platform entity relationships (offerings, licenses, features).
- `monaiq://config/endpoints` — Canonical endpoint URLs.

## Utility Workflows

When state detection shows SDK is already integrated, offer these options:

- **Credential recovery** — Re-read the completed checkout result or the application's credential store to recover the purchased EncodedCredential. The profile tool only returns reseller checkout credentials.
- **Update service URIs** — Modify licensing configuration to point to different environments (staging, production). Resolve `monaiq://config/endpoints` for the current authoritative URIs.
- **Switch credential source** — Migrate from configuration-based to user-managed credentials (or vice versa). Involves implementing or removing a custom credential resolver; resolve `monaiq://sdk/{stack}/setup` for the migration narrative.
- **Verify integration** — Run a quick health check: confirm packages are installed, configuration is present, DI is registered, and a test authorization call succeeds.

</reference>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Package install fails | NuGet/npm error during SDK package installation | Check network connectivity and package source configuration. Retry with an explicit registry/source (`--source nuget.org` for .NET, clear npm cache for React). Resolve `monaiq://sdk/{stack}/setup` for the exact package name and commands. |
| Credential retrieval fails | `profile` tool returns an error | Verify the session is active via `register_or_login`. Re-authenticate if the session expired. |
| Config binding fails | Licensing configuration exception at startup | Verify the configuration section name and key casing match the platform-specific configuration shape. Check that the credential field is not empty or malformed. Resolve `monaiq://sdk/{stack}/setup` for the authoritative configuration keys. |
| DI registration fails | Build error on the SDK's registration extension or provider component | Verify the package is installed and the correct imports/usings are present. Resolve `monaiq://platforms/api-surface/{platform}` for the authoritative extension-method or component signatures. |
| Authorization call returns null | License validation fails at runtime | Verify the purchased EncodedCredential is present in configuration or the application's credential store. Resolve `monaiq://config/endpoints` to confirm service URIs are correct for the target environment. Resolve `monaiq://platforms/pitfalls/{platform}` for platform-specific null-semantics differences. |

SDK integration steps are non-destructive — each step modifies source files that can be edited again safely.
</error-recovery>

<brownfield>
## Brownfield Credential Contract

Persist the EncodedCredential (brownfield) only after an ownership scope decision. Before storage advice or schema changes, stop at `CHECKPOINT-PRE-BROWNFIELD-MIGRATION` and choose one scope: app-wide configuration credential, tenant-level credential, or user-level credential. The selected scope determines where the application reads the credential, who can update it, and which privacy boundary applies.

For existing apps, prefer additive nullable changes that preserve current reads and allow a null/unlicensed rollout state. Do not make existing users fail because a credential column, tenant setting, or config value is absent during rollout. Plan a separate backfill for existing customers, define rollback and recovery steps, and keep old code paths readable until the migration is verified.

Use approved local secret or configuration stores for app-wide credentials, tenant storage for tenant-level credential scope, and user/profile storage for user-level credential scope. Never place raw ApiKey, EncodedCredential, `.env`, user-secrets, or secret-bearing values in prompts, journals, docs, generated plugin output, or client-visible code/config. Use placeholders and resource-backed instructions instead.

For checkout or license delivery, reliable email remains a prerequisite. If the existing user/customer/tenant table lacks a trustworthy email, pause before purchase-flow code and decide how the host app will collect or verify it.

For rate-limit defaults during migration, preserve unlimited and capped semantics. Do not accidentally introduce low capped limits while backfilling or assigning features; make quota, reset cadence, and recovery behavior explicit.

For platform-specific persistence pitfalls (web storage semantics, lifecycle on desktop shells), resolve:

Fetch `monaiq://platforms/pitfalls/{platform}` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.
</brownfield>

<success_criteria>
- Project builds without compilation errors after SDK integration.
- The SDK's authorization call returns a non-null result at runtime.
- No licensing configuration exception in application logs.
- Correct credential source pattern used (Configuration-Based or User-Managed).
- All namespaces match the `monaiq://domain/namespaces` reference.
</success_criteria>
