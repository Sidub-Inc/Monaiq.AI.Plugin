---
name: domain-reference
description: "Use when: explaining Monaiq domain concepts, entity relationships, FeatureKey semantics, namespace locations, products, features, offerings, licenses, credentials, or checkout terminology."
agent: monaiq
auto-invoke:
  - "User asks about Monaiq entity relationships or domain concepts"
  - "User wants to understand the licensing domain model"
  - "User asks what a FeatureKey, ProductOffering, or LicenseFeature is"
tags: [domain, model, knowledge, reference]
category: domain
allowed-tools: [Read, fetch_step_resources, monaiq_journal, mcp__plugin_monaiq_monaiq__fetch_step_resources, mcp__plugin_monaiq_monaiq__monaiq_journal]
argument-hint: "topic (entities|features|offerings|licenses)"
tier: 3
invoked-by: [getting-started, manage-catalog, implement-licensing]
---

<objective>
Answer Monaiq domain questions from authoritative MCP resources, translating user vocabulary into domain concepts without starting a mutation workflow unless the user asks for consequential work.
</objective>

<input-context>
Receives from any upstream skill:
- topic (optional): specific domain concept to explain (entities, features, offerings, licenses)
- userVocabulary (optional): the plain-language term the user used — map to domain equivalent

If invoked without upstream context, asks the user what they want to understand.
</input-context>

<output-context>
Provides to requesting skill:
- domainContext: { entities: relevant entity descriptions, relationships: key connections, fieldReference: critical field definitions }

This is a support skill — invoked on-demand by other skills, not part of a linear chain.
</output-context>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (read-only variant). Pivot to the mutation-capable variant the moment the user asks for catalog, credential/config, or app-behavior changes.
</monaiq-agent-handoff>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This is a read-only reference skill; it answers domain questions without journal startup unless the user pivots to consequential work.
</execution_context>

<workflow>
1. Classify the user's question as entity, relationship, field meaning, namespace/API surface, checkout/credential terminology, or follow-up implementation work.
2. Fetch only the resources needed: `monaiq://domain/model` for entities/relationships, `monaiq://domain/namespaces` for type locations, and platform API surface only when the question asks for implementation signatures.
3. Answer the specific question in business-readable terms first, then include compact technical backing for agents when useful. Do not paste resource content wholesale.
4. If the user pivots from explanation into catalog mutation, credential/config writes, or app behavior changes, stop and route through the full journal/checkpoint workflow for the relevant specialist skill.
</workflow>

<reference>
## Topic Routing Reference

- Entity question -> resolve `monaiq://domain/model` and focus on that entity.
- Namespace question -> resolve `monaiq://domain/namespaces`.
- Relationship question -> resolve both resources and explain the connection.
- Implementation signature question -> route or fetch platform API surface before answering.

## Fetch Domain Model

Fetch the domain model MCP resource. This contains comprehensive documentation of all entities, their relationships, key fields, polymorphic hierarchies, and invariants.

Fetch `monaiq://domain/model` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

**Key entities covered:**
- Customer, Product, ProductFeature (polymorphic: Access/RateLimit)
- ProductOffering, FeatureOffering (polymorphic: ServiceAccess/RateLimit)
- License, LicenseFeature, BillingPlan
- KeyDescriptor, EncodedCredential

## Fetch Namespace Reference

Fetch the namespace reference MCP resource. This provides authoritative type-to-namespace mappings for .NET and React, ensuring code references use the correct imports.

Fetch `monaiq://domain/namespaces` via the MCP `resources/read` operation or `fetch_step_resources` tool before proceeding.

## Answer the User's Question

Use the fetched resources as internal context to answer the user's specific domain question. Provide a focused answer about the entities, relationships, or concepts they asked about.

**Common question patterns:**

| Question Type | Key Entities | Critical Fields |
|--------------|-------------|-----------------|
| "What is a FeatureKey?" | ProductFeature, FeatureOffering, LicenseFeature | `FeatureKey` — the linking field across catalog and runtime |
| "How do features work?" | ProductAccessFeature, ProductRateLimitFeature | Feature flags (Access) and usage limits (RateLimit) — polymorphic types with distinct assertion classes |
| "What's the difference between offerings and licenses?" | ProductOffering, License | Pricing tier (Offering) = pricing template, License = issued entitlement |
| "How does checkout connect to licensing?" | CheckoutRequest, License, EncodedCredential | CorrelationId flow, credential provisioning |
| "What types are in which namespace?" | All entities | Use `monaiq://domain/namespaces` reference |

</reference>

<success_criteria>
- Domain model resource (`monaiq://domain/model`) was fetched and used as context
- Answer addresses the user's specific domain question accurately
- Entity relationships and field descriptions match the authoritative resource content
- Namespace references (if relevant) match the `monaiq://domain/namespaces` resource
</success_criteria>
