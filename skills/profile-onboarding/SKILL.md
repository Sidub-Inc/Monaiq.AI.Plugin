---
name: profile-onboarding
description: "Use when: viewing Monaiq reseller profile status, retrieving reseller ApiKey and IssuerClientId, or reviewing terms and privacy documents."
agent: monaiq
auto-invoke:
  - "User wants to view their reseller profile or retrieve SDK credentials"
  - "User wants to review terms of service before accepting"
  - "User asks about their onboarding status or account details"
tags: [profile, onboarding, credentials, terms]
category: onboarding
allowed-tools: [register_or_login, profile, monaiq_journal, fetch_step_resources, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__monaiq_journal, mcp__plugin_monaiq_monaiq__fetch_step_resources]
tier: 3
invoked-by: [getting-started]
---

<objective>
Review reseller profile status, retrieve reseller checkout credentials when requested, and present terms/privacy documents without leaking secret-bearing values. Profile acceptance is a checkpointed tool action, not automatic background work.
</objective>

<input-context>
Receives from getting-started:
- detectedState (optional): { profileComplete, resellerEnabled } — skip viewing if state is known

If invoked without upstream context, checks profile state directly.
</input-context>

<output-context>
Provides to downstream skills:
- profileData: { issuerClientId, profileStatus, resellerStatus, apiKeyAvailable }

Used by implement-purchase-flow for checkout configuration. Purchased EncodedCredential values come from checkout results, not the reseller profile.
</output-context>

<execution_context>
Follows the skill layout and shared workflows in `_shared/protocols.md`. This skill contributes only session/profile status, terms review, credential availability summaries, and checkpointed terms acceptance guidance.
</execution_context>

<monaiq-agent-handoff>
Follow the Direct Invocation Contract in `_shared/protocols.md` (mutation-capable variant — terms acceptance is a checkpointed tool action).
</monaiq-agent-handoff>

<workflow>
1. Run `_shared/workflows/startup.md` for `profile-onboarding`.
2. **Response Pattern.** Follow `_shared/protocols.md` § Response Pattern. The gate this skill owns is **profile status + terms acceptance**. Evidence sources, in priority order: `register_or_login` session result, `profile` (`ProfileStatus`, `ResellerStatus`, `IssuerClientId`), prior journal decisions, route packet. The recommendation is the next forward action that unblocks catalog/checkout/implementation; the host-native question lets the user proceed, view terms/privacy, or pause.
3. Establish a session with `register_or_login`, then call `profile` to detect `ProfileStatus`, `ResellerStatus`, and `IssuerClientId`.
3. Present status in business-readable terms: what the account can do now, what is blocked, and which next action unblocks catalog, checkout, or implementation work.
4. Retrieve reseller credentials only when needed for catalog/checkout setup or explicitly requested. Never write raw `ApiKey`, `EncodedCredential`, `.env`, user-secrets, or secret-bearing values into prompts, `.monaiq`, summaries, docs, or generated plugin output.
5. If the user wants to accept terms, stop at `CHECKPOINT-PRE-TERMS-ACCEPTANCE`, follow `_shared/workflows/checkpoint.md`, present the terms/privacy review state, record the user's approval result, then call `profile` step 4 only after approval.
6. Coalesce profile status summaries, checklist progress, and handoff context through `_shared/workflows/completion.md` without secret values, then call `skill_completed` once.
</workflow>

<reference>
## State Routing Reference

- ProfileStatus = NotStarted -> guide full profile setup.
- ProfileStatus = Incomplete -> show missing profile work.
- ProfileStatus = Completed and ResellerStatus = Pending -> explain approval wait state.
- ResellerStatus = Enabled -> show summary and offer credential retrieval.

## View Profile

Call the `profile` tool with `startStep=1` to view the reseller profile.

**Key fields:**

| Field | Description |
|-------|-------------|
| `IssuerClientId` | Unique reseller identifier — used in SDK configuration and checkout requests |
| `ContactEmail` | Primary contact email for the account |
| `LegalName` | Legal business name on file |
| `ProfileStatus` | Account completeness: `NotStarted` → `Incomplete` → `Completed` |
| `ResellerStatus` | Onboarding state: `Disabled` → `Pending` → `Enabled` |

**Status meanings:**

- **ProfileStatus = NotStarted** — Account created but no profile information submitted
- **ProfileStatus = Incomplete** — Some profile fields are filled but required information is missing
- **ProfileStatus = Completed** — All required profile information is on file
- **ResellerStatus = Disabled** — Reseller features not activated; cannot create products or offerings
- **ResellerStatus = Pending** — Reseller application submitted, awaiting approval
- **ResellerStatus = Enabled** — Full reseller access; can create products, offerings, and issue licenses

The `IssuerClientId` is the primary identifier for the reseller account. It is used as the `IssuerClientId` parameter in checkout requests and appears in license metadata.

## Retrieve Credentials

Call the `profile` tool with `startStep=2` to retrieve reseller credentials used for catalog and checkout operations.

**Credential fields:**

| Credential | Purpose | Where Used |
|-----------|---------|------------|
| `ApiKey` | Authenticates checkout API calls | `CreateCheckoutSession` and `GetCheckoutResult` API key parameter |
| `IssuerClientId` | Reseller identity for checkout requests | `CheckoutRequest.IssuerClientId` |

EncodedCredential is not a reseller profile credential. It is produced after an offering is purchased and returned by checkout-result retrieval.

**Security:** Do not persist the `ApiKey` to disk or commit it to source control. Use environment variables or a secrets manager for production deployments.
Do not write raw `ApiKey`, `IssuerClientId` plus secret context, `EncodedCredential`, `.env`, or user-secret values into prompts, `.monaiq`, summaries, or generated plugin output.

**How credentials connect to checkout setup:**

- `ApiKey` → Used when calling `ICheckoutService` methods (see the `implement-purchase-flow` skill)
- `IssuerClientId` → Used in `CheckoutRequest` for embedded purchases

## Read Terms

Call the `profile` tool with `startStep=3` to read the Terms of Service and Privacy Policy.

Both documents must be presented to the user before acceptance. The terms cover:

- **Terms of Service** — Account registration, use of services, intellectual property, data processing, limitation of liability
- **Privacy Policy** — Data collection, usage, storage practices, user rights under applicable privacy laws

Review both documents carefully before proceeding to accept terms.

## Accepting Terms (Step 4)

**Important:** Accepting terms is NOT part of this skill workflow. It is a **tool action** that modifies state.

To accept terms, call the `profile` tool directly with `startStep=4` and `data={"customerTerms": true, "resellerTerms": true}`. Both `customerTerms` and `resellerTerms` must be set to `true` to complete terms acceptance. This action records the user's agreement and updates the profile status.

## Related Tools

- `register_or_login` — Establish a session (prerequisite for all profile operations)
- `profile` — The tool that executes each step of this workflow
- `implement_base` — SDK integration (configures where purchased credentials are supplied at runtime)
- `implement_purchase_flow` — Checkout integration (uses ApiKey and IssuerClientId from step 2)

## Utility Workflows

- **Credential recovery**: Re-retrieve reseller checkout credentials from your profile. Use purchase-flow checkout results or your application storage for purchased `EncodedCredential` values.
- **View terms**: Review Terms of Service and Privacy Policy before accepting.
- **Check approval status**: See current ResellerStatus and what to expect next.

</reference>

<success_criteria>
- Profile information is visible including `IssuerClientId`, `ProfileStatus`, and `ResellerStatus`
- Reseller credentials (`ApiKey`, `IssuerClientId`) are retrieved successfully
- Terms of Service and Privacy Policy are presented for review
- Step 4 (terms acceptance) is understood as a separate tool action with `startStep=4`
- No raw `ApiKey`, `EncodedCredential`, `.env`, or user-secret values are written into prompts, `.monaiq`, summaries, or generated plugin output
</success_criteria>
