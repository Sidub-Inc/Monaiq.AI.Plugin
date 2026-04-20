---
name: profile-onboarding
description: View your reseller profile, retrieve your license key and API credentials, and review terms of service
auto-invoke:
  - "User wants to view their reseller profile or retrieve SDK credentials"
  - "User wants to review terms of service before accepting"
  - "User asks about their onboarding status or account details"
tags: [profile, onboarding, credentials, terms]
category: onboarding
allowed-tools: [register_or_login, profile, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile]
tier: 3
invoked-by: [getting-started]
---

<input-context>
Receives from getting-started:
- detectedState (optional): { profileComplete, resellerEnabled } — skip viewing if state is known

If invoked without upstream context, checks profile state directly.
</input-context>

<output-context>
Provides to downstream skills:
- profileData: { issuerClientId, profileStatus, resellerStatus, encodedCredential (if retrieved) }

Used by implement-licensing for credential setup and by implement-purchase-flow for checkout configuration.
</output-context>

<state-detection>
Before showing profile:
1. Call `profile` tool to get current status
2. Check ProfileStatus and ResellerStatus

Based on state:
- ProfileStatus = NotStarted → Guide through full profile setup
- ProfileStatus = Incomplete → Show what's missing, guide completion
- ProfileStatus = Completed, ResellerStatus = Pending → Show status, explain approval process
- Everything complete → Show summary, offer credential retrieval
</state-detection>

<objective>
Guide an agent through reviewing a reseller profile, retrieving SDK credentials, and reading the Terms of Service and Privacy Policy. This covers three read-only steps. Terms acceptance (step 4) is a separate tool action — not part of this skill workflow.
</objective>

<process>

## Prerequisites

- Establish a session using the `register_or_login` tool

## Step 1: View Profile

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

## Step 2: Retrieve Credentials

Call the `profile` tool with `startStep=2` to retrieve SDK credentials.

**Credential fields:**

| Credential | Purpose | Where Used |
|-----------|---------|------------|
| `ApiKey` | Authenticates checkout API calls | `CreateCheckoutSession` and `GetCheckoutResult` API key parameter |
| `EncodedCredential` | Encoded string containing license ID, service key, and access key | `LicensingServiceOptions.EncodedCredential` in SDK configuration |
| `IssuerClientId` | Reseller identity for checkout requests | `CheckoutRequest.IssuerClientId` |

**Security:** Do not persist the `ApiKey` to disk or commit it to source control. Use environment variables or a secrets manager for production deployments.

**How credentials connect to SDK setup:**

- `EncodedCredential` → Goes into `LicensingServiceOptions` (see the `implement-licensing` skill)
- `ApiKey` → Used when calling `ICheckoutService` methods (see the `implement-purchase-flow` skill)
- `IssuerClientId` → Used in `CheckoutRequest` for embedded purchases

## Step 3: Read Terms

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
- `implement_base` — SDK integration (uses credentials from step 2)
- `implement_purchase_flow` — Checkout integration (uses ApiKey and IssuerClientId from step 2)

## Utility Workflows

- **Credential recovery**: Re-retrieve your license key (`EncodedCredential`) from your profile. Call `profile` tool with step 2 to see credentials.
- **View terms**: Review Terms of Service and Privacy Policy before accepting.
- **Check approval status**: See current ResellerStatus and what to expect next.

</process>

<success_criteria>
- Profile information is visible including `IssuerClientId`, `ProfileStatus`, and `ResellerStatus`
- SDK credentials (`ApiKey`, `EncodedCredential`) are retrieved successfully
- Terms of Service and Privacy Policy are presented for review
- Step 4 (terms acceptance) is understood as a separate tool action with `startStep=4`
</success_criteria>
