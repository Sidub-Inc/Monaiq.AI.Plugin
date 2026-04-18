---
name: implement-purchase-flow
description: Add a buy button to your app — integrate Stripe checkout so users can purchase licenses without leaving your application
auto-invoke:
  - "User wants to add in-app purchases or checkout to their application"
  - "User wants to integrate Stripe checkout for license purchasing"
  - "User asks how to embed a buy button or purchase flow"
tags: [sdk, checkout, purchase, licensing, stripe]
category: integration
allowed-tools: [offering, feature_offering, profile, implement_purchase_flow]
argument-hint: "language (dotnet|react)"
tier: 3
invoked-by: [implement-licensing, manage-catalog]
---

<input-context>
Receives from implement-licensing or manage-catalog:
- sdkConfig (optional): { language, credentialSource } — confirms SDK is ready
- catalogSpec (optional): { offerings: [{ code, classification, baseRate }] } — available offerings to sell

If invoked without upstream context, verifies SDK integration and discovers offerings from catalog.
</input-context>

<output-context>
Provides completion context:
- checkoutImpl: { architecture: "server-initiated" | "client-only", offeringCodes: [], checkoutRoute: string, successHandler: string }

This skill is a leaf node — no downstream skills depend on it.
</output-context>

<state-detection>
Before implementing checkout:
1. Verify SDK is integrated — look for licensing package references
2. Call `offering` (list) — check for published offerings to sell
3. Check for existing checkout implementation — look for checkout session creation code

Based on detected state:
- SDK not integrated → Route to implement-licensing first
- No offerings exist → Route to manage-catalog to create offerings
- Offerings exist but all Draft → Warn that offerings need to be published (Status = Public) for checkout
- Existing checkout code found → Show current implementation, offer modifications
</state-detection>

<objective>
Guide an agent through adding an embedded purchase flow to a .NET or React application so users can buy licenses without leaving the app. Covers four areas: offering discovery and architecture decisions, backend checkout session creation, success handling with credential provisioning, and context provider wiring.
</objective>

<process>

## Prerequisites

- Licensing SDK already integrated (complete the `implement-licensing` skill first)
- Establish a session using the `register_or_login` tool
- Retrieve credentials via the `profile` tool (step 2) — you need your reseller account identifier (`IssuerClientId`) and `ApiKey`
- Published offerings exist — use the `offering` and `feature_offering` tools to browse the catalog
- Fetch the SDK setup guide from `monaiq://sdk/{language}/setup`

## Step 1: Discovery

Determine your application's checkout architecture and identify the offerings to sell.

**Architecture decision:**

| Architecture | Flow | Recommendation |
|-------------|------|----------------|
| Has backend (API, BFF, SSR) | Server-Initiated Checkout | Recommended — secure, correlation ID stays server-side |
| Frontend only (SPA) | Client-Only Checkout | Viable — ensure HTTPS, correlation ID managed client-side |

**CorrelationId** is a tracking identifier (called `CorrelationId`) that links the purchase back to the buyer. It's an opaque string YOUR application provides that round-trips through the entire checkout flow:

1. Your app sends CorrelationId with the checkout request
2. Stripe Checkout completes, webhook fires, license is created
3. `GetCheckoutResult` returns CorrelationId + the license key (`EncodedCredential`)
4. Your app matches CorrelationId to the user and stores the credential

Use a stable, unique identifier: user ID, tenant ID, or a composite key.

**Browse offerings** using the `offering` tool to see what's available for purchase. Free, trial, and zero-dollar offerings are fully supported — the API handles them without a Stripe redirect.

## Step 2: Backend Integration

Create a checkout session using the SDK's `ICheckoutService`.

### .NET

```csharp
using Sidub.Licensing.Checkout;
using Sidub.Licensing.Services;

public class CheckoutController
{
    private readonly ICheckoutService _checkoutService;

    public CheckoutController(ICheckoutService checkoutService)
    {
        _checkoutService = checkoutService;
    }

    public async Task<string> CreateCheckoutAsync(
        Guid offeringId, string correlationId, string customerEmail)
    {
        var serviceReference = /* your LicensingServiceReference */;
        var apiKey = "your-api-key"; // from profile tool step 2

        var request = new CheckoutRequest
        {
            OfferingId = offeringId,
            IssuerClientId = /* your IssuerClientId from profile */,
            CorrelationId = correlationId,
            CustomerEmail = customerEmail,
            SuccessUrl = "https://yourapp.com/checkout/success?session={CHECKOUT_SESSION_ID}",
            CancelUrl = "https://yourapp.com/checkout/cancel"
        };

        var session = await _checkoutService.CreateCheckoutSession(
            serviceReference, request, apiKey);

        // session.SessionUrl is the Stripe Checkout URL (null for free/trial)
        // session.SessionId is used to poll for results
        return session.SessionUrl ?? "/checkout/success";
    }
}
```

### React

```tsx
import { useLicensingContext } from '@sidub-inc/licensing-client';

async function startCheckout(client, offeringId, correlationId, email) {
  const session = await client.createCheckoutSession({
    offeringId,
    issuerClientId: 'your-issuer-client-id',
    correlationId,
    customerEmail: email,
    successUrl: `${window.location.origin}/checkout/success?session={CHECKOUT_SESSION_ID}`,
    cancelUrl: `${window.location.origin}/checkout/cancel`
  }, 'your-api-key');

  if (session.sessionUrl) {
    window.location.href = session.sessionUrl; // redirect to Stripe
  } else {
    // Free/trial — no Stripe redirect needed
    await handleSuccess(client, session.sessionId);
  }
}
```

**Security notes:**
- Never expose the `ApiKey` to frontend code in production — proxy through your backend
- `IssuerClientId` identifies your reseller account — obtained from the `profile` tool
- Use HTTPS for all `SuccessUrl` and `CancelUrl` values

## Step 3: Success Handling

After the user completes Stripe Checkout, poll for the result using the session ID.

### .NET

```csharp
public async Task<CheckoutSessionResult> HandleSuccessAsync(string sessionId)
{
    var serviceReference = /* your LicensingServiceReference */;
    var apiKey = "your-api-key";

    var result = await _checkoutService.GetCheckoutResult(
        serviceReference, sessionId, apiKey);

    // result.Status — checkout status
    // result.CorrelationId — matches what you sent
    // result.EncodedCredential — the license credential string
    // result.LicenseId — the issued license ID

    // Store the credential for the user identified by CorrelationId
    await StoreCredentialForUser(result.CorrelationId, result.EncodedCredential);

    return result;
}
```

### React

```tsx
async function handleSuccess(client, sessionId) {
  const result = await client.getCheckoutResult(sessionId, 'your-api-key');

  // result.correlationId — matches your user
  // result.encodedCredential — store this for the user
  // result.licenseId — the issued license

  localStorage.setItem('sidub_credential', result.encodedCredential);
}
```

## Step 4: Context Provider Wiring

Connect the purchased credential to the licensing SDK so feature checks work at runtime.

**For User-Managed credentials** (multi-tenant / per-user licenses):

### .NET

Implement `ILicensingContextProvider` to resolve from your user storage:

```csharp
using Sidub.Licensing.Context;

public class UserLicensingContextProvider : ILicensingContextProvider
{
    private readonly IUserService _userService;

    public UserLicensingContextProvider(IUserService userService)
    {
        _userService = userService;
    }

    public async Task<LicensingContext?> ResolveContextAsync(
        CancellationToken cancellationToken = default)
    {
        var credential = await _userService.GetStoredCredentialAsync(cancellationToken);

        if (credential is null)
            return null; // pre-licensed state — show purchase UI

        return LicensingContext.FromEncodedString(credential);
    }
}
```

Register in `Program.cs`:

```csharp
builder.Services.AddSidubLicensing<UserLicensingContextProvider>();
```

### React

Update `LicensingProvider` to use the stored credential:

```tsx
function App() {
  const credential = localStorage.getItem('sidub_credential');

  return (
    <LicensingProvider config={{
      licenseServiceUri: 'https://api.monaiq.com/licensing',
      consumptionServiceUri: 'https://licensing-consumption.sidub.ca',
      encodedCredential: credential
    }}>
      <YourApp />
    </LicensingProvider>
  );
}
```

**For Configuration-Based** (single organization license): Store the `EncodedCredential` from the checkout result in your app settings and restart — the default `ConfigurationLicensingContextProvider` handles the rest.

## Verification

1. Create a checkout session with a test offering
2. Complete the Stripe Checkout flow (or verify free/trial auto-completes)
3. Poll `GetCheckoutResult` and confirm `EncodedCredential` is returned
4. Store the credential and verify `GetAuthorization()` returns a valid result
5. Verify feature checks work with the new license

## Related Tools

- `implement_purchase_flow` — Interactive step-by-step checkout integration guidance
- `offering` — Browse available product offerings
- `feature_offering` — View feature assignments for an offering
- `profile` — Retrieve `IssuerClientId` and `ApiKey` credentials

## Related Resources

- `monaiq://sdk/dotnet/setup` — .NET SDK setup
- `monaiq://sdk/react/setup` — React SDK setup
- `monaiq://domain/model` — Entity relationships (Offerings, Licenses)

</process>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Checkout session creation fails | `CreateCheckoutSession` returns error | Verify `ApiKey` and `IssuerClientId` are correct (retrieve from `profile` tool). Check that the offering exists and has Status = Public. |
| Stripe redirect fails | User sees Stripe error page | Verify `SuccessUrl` and `CancelUrl` use HTTPS and include the `{CHECKOUT_SESSION_ID}` placeholder. Check that the offering's `BaseRate` and `Currency` are valid. |
| GetCheckoutResult returns no credential | Poll returns empty `EncodedCredential` | The checkout may not be complete yet — retry polling after a short delay. If persistent, verify the session ID is correct. |
| Credential storage fails | License purchased but credential lost | Re-call `GetCheckoutResult` with the same session ID to retrieve the credential again. Session results are persistent. |
| Context provider wiring fails | Feature checks return null after purchase | Verify `ILicensingContextProvider` implementation returns the stored credential. Check DI registration uses `AddSidubLicensing<T>()` with your custom provider. |

Checkout sessions are idempotent — creating a new session for the same offering is safe. Completed purchases are recorded server-side and can be recovered.
</error-recovery>

<success_criteria>
- Checkout session creation works — returns a Stripe URL (paid) or null (free/trial)
- `GetCheckoutResult` returns `EncodedCredential` and `CorrelationId` after payment completes
- Credential is stored and `GetAuthorization()` returns a valid result using the purchased license
- Feature checks work at runtime with the purchased credential
- `ApiKey` is not exposed to frontend code in production
</success_criteria>
