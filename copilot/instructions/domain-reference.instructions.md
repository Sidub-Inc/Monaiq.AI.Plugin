---
name: domain-reference
description: Domain knowledge bridge with context flow — answers questions about licensing concepts by fetching the Monaiq domain model
auto-invoke:
  - "User asks about Monaiq entity relationships or domain concepts"
  - "User wants to understand the licensing domain model"
  - "User asks what a FeatureKey, ProductOffering, or LicenseFeature is"
tags: [domain, model, knowledge, reference]
category: domain
allowed-tools: []
argument-hint: "topic (entities|features|offerings|licenses)"
tier: 3
invoked-by: [getting-started, manage-catalog, implement-licensing]
---

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

<state-detection>
Before answering:
1. Check if the user's question maps to a specific entity or concept
2. Determine if the question is about entities, relationships, field meanings, or namespace locations

Based on detected topic:
- Entity question → Fetch monaiq://domain/model, focus response on that entity
- Namespace question → Fetch monaiq://domain/namespaces
- Relationship question → Fetch both resources, explain the connection
</state-detection>

<objective>
Answer questions about Monaiq domain concepts by fetching the domain model and namespace reference MCP resources. The resource content is agent-facing context — use it to inform your response, not to surface directly to the user.
</objective>

<process>

## Step 1: Fetch Domain Model

Fetch the `monaiq://domain/model` MCP resource. This contains comprehensive documentation of all entities, their relationships, key fields, polymorphic hierarchies, and invariants.

**Key entities covered:**
- Customer, Product, ProductFeature (polymorphic: Access/RateLimit)
- ProductOffering, FeatureOffering (polymorphic: ServiceAccess/RateLimit)
- License, LicenseFeature, BillingPlan
- KeyDescriptor, EncodedCredential

## Step 2: Fetch Namespace Reference

Fetch the `monaiq://domain/namespaces` MCP resource. This provides authoritative type-to-namespace mappings for .NET and React, ensuring code references use the correct imports.

## Step 3: Answer the User's Question

Use the fetched resources as internal context to answer the user's specific domain question. Provide a focused answer about the entities, relationships, or concepts they asked about.

**Common question patterns:**

| Question Type | Key Entities | Critical Fields |
|--------------|-------------|-----------------|
| "What is a FeatureKey?" | ProductFeature, FeatureOffering, LicenseFeature | `FeatureKey` — the linking field across catalog and runtime |
| "How do features work?" | ProductAccessFeature, ProductRateLimitFeature | Feature flags (Access) and usage limits (RateLimit) — polymorphic types with distinct assertion classes |
| "What's the difference between offerings and licenses?" | ProductOffering, License | Pricing tier (Offering) = pricing template, License = issued entitlement |
| "How does checkout connect to licensing?" | CheckoutRequest, License, EncodedCredential | CorrelationId flow, credential provisioning |
| "What types are in which namespace?" | All entities | Use `monaiq://domain/namespaces` reference |

</process>

<success_criteria>
- Domain model resource (`monaiq://domain/model`) was fetched and used as context
- Answer addresses the user's specific domain question accurately
- Entity relationships and field descriptions match the authoritative resource content
- Namespace references (if relevant) match the `monaiq://domain/namespaces` resource
</success_criteria>
