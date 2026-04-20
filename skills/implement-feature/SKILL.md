---
name: implement-feature
description: Add feature gates to your app — implement license checks for premium features (on/off) and usage limits (metered quotas)
auto-invoke:
  - "User wants to add feature gating to their application"
  - "User wants to implement license feature checks"
  - "User asks how to gate functionality by license features"
tags: [sdk, features, licensing, entitlements, access, ratelimit]
category: integration
allowed-tools: [product, product_feature, implement_product_feature, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__product_feature, mcp__plugin_monaiq_monaiq__implement_product_feature]
argument-hint: "featureKey, featureType (access|ratelimit)"
tier: 3
invoked-by: [implement-licensing, manage-catalog]
---

<input-context>
Receives from implement-licensing:
- sdkConfig: { language, credentialSource, serviceOptionsConfigured, diRegistered } — confirms SDK is ready
- targetFeatures (optional): [{ featureKey, featureType }] — pre-selected features to implement

If invoked without upstream context, verifies SDK integration exists and discovers features from catalog.
</input-context>

<output-context>
Provides to downstream skills:
- featureImpl: { implementedFeatures: [{ featureKey, featureType, codeFile }], buildVerified: boolean }

This is typically the last step in the implementation chain:
getting-started → manage-catalog → implement-licensing → implement-feature
</output-context>

<state-detection>
Before implementing feature checks:
1. Verify SDK is integrated — look for licensing package references and DI registration
2. Call `product_feature` (list) — get available features and their types
3. Check existing code for feature assertions — look for existing `ServiceAccessLicenseAssertion` or `RateLimitLicenseAssertion` calls

Based on detected state:
- SDK not integrated → Route to implement-licensing first
- No features in catalog → Route to manage-catalog to create features
- Features exist, none implemented → Full flow from Step 1
- Some features already implemented → Show which are done, offer to implement remaining
</state-detection>

<objective>
Guide an agent through adding license feature checks to a .NET or React application. Features are polymorphic — **feature flags** (Access — binary gate: allowed or denied) and **usage limits** (RateLimit — metered consumption tracking). Covers feature discovery, implementation of the correct assertion pattern, and build verification.
</objective>

<process>

## Prerequisites

- Licensing SDK already integrated (complete the `implement-licensing` skill first)
- Product and features exist in the catalog — use the `product` and `product_feature` tools to verify
- Fetch the domain model from `monaiq://domain/model` to understand entity relationships
- Fetch the namespace reference from `monaiq://domain/namespaces` for type locations

## Step 1: Discover Feature

Identify the feature to gate and determine its type.

**Use the `product_feature` tool** to list features for a product. Each feature has:

| Field | Purpose |
|-------|---------|
| `FeatureKey` | The feature key (the unique identifier you defined in your catalog) used in code to reference this feature |
| `Kind` | `ServiceAccess` or `RateLimit` — determines the assertion pattern |
| `DisplayName` | Human-readable name |
| `ServiceType` | (Access only) The service type classification |

**Feature Type Comparison:**

| Aspect | Feature Flag (Access) | Usage Limit (RateLimit) |
|--------|---------------------|---------------------|
| Pattern | Assert → allowed/denied | Get feature → record consumption → assert |
| Assertion class | `ServiceAccessLicenseAssertion` | `RateLimitLicenseAssertion` |
| Key types | `ServiceAccessLicenseFeature` | `RateLimitLicenseFeature`, `RateLimitLicenseFeatureOperation` |
| Use case | Premium content, feature flags | API rate limits, usage quotas |

**Critical:** Each feature type has its own specific assertion class. Mixing types causes runtime failures — a feature flag check (using `ServiceAccessLicenseAssertion`) for Access features, a usage limit check (using `RateLimitLicenseAssertion`) for RateLimit features.

## Step 2: Implement Feature Check

### Feature Flag Check (Access — Binary Gate)

#### .NET

```csharp
using Sidub.Licensing;
using Sidub.Licensing.Services;
using Sidub.Licensing.Features;

public async Task<bool> CheckFeatureAccessAsync(string featureKey)
{
    var assertion = ServiceAccessLicenseAssertion.Create(featureKey);
    var result = await _licensingService.AssertLicense(assertion);
    return result.IsAllowed;
}
```

#### React

```tsx
import { useLicensingContext } from '@sidub-inc/licensing-client';
import { ServiceAccessAssertion } from '@sidub-inc/licensing-client';

function FeatureGate({ featureKey, children }) {
  const client = useLicensingContext();
  const [allowed, setAllowed] = useState(null);

  useEffect(() => {
    async function check() {
      const auth = await client.getAuthorization();
      const result = await client.assertLicense(
        ServiceAccessAssertion.create(featureKey), auth
      );
      setAllowed(result.isAllowed);
    }
    check();
  }, [client, featureKey]);

  if (allowed === null) return <div>Loading...</div>;
  if (!allowed) return <div>Upgrade to access this feature.</div>;
  return <>{children}</>;
}
```

### Usage Limit Check (RateLimit — Metered Consumption)

Usage limit features (RateLimit) follow a multi-step pattern: retrieve the feature, record consumption, then assert the limit.

#### .NET

```csharp
using Sidub.Licensing;
using Sidub.Licensing.Services;
using Sidub.Licensing.Features;

public async Task<bool> ConsumeAndCheckAsync(string featureKey, int amount)
{
    // 1. Get the feature to obtain its current state
    var feature = await _licensingService.GetLicenseFeature<RateLimitLicenseFeature>(featureKey);

    // 2. Record consumption
    var operation = new RateLimitLicenseFeatureOperation(feature, amount);
    await _licensingService.RecordConsumption(operation);

    // 3. Assert the limit
    var assertion = RateLimitLicenseAssertion.Create(featureKey);
    var result = await _licensingService.AssertLicense(assertion);
    return result.IsAllowed;
}
```

#### React

```tsx
import { useLicensingContext } from '@sidub-inc/licensing-client';

async function consumeFeature(client, featureKey, amount) {
  const auth = await client.getAuthorization();

  // Record consumption
  await client.recordConsumption(featureKey, amount, auth);

  // Check if still within limits
  const assertion = RateLimitAssertion.create(featureKey);
  const result = await client.assertLicense(assertion, auth);
  return result.isAllowed;
}
```

## Step 3: Build and Verify

1. **Build** — Compile the project and resolve any missing namespace imports using `monaiq://domain/namespaces`
2. **Test the gate** — Invoke the feature check and verify:
   - Access features return `IsAllowed = true` when the license grants access
   - RateLimit features return `IsAllowed = true` when consumption is within limits
3. **Test the deny path** — Verify your application handles the denied case gracefully (upgrade prompt, fallback behavior, etc.)

## Related Tools

- `implement_product_feature` — Interactive step-by-step feature integration guidance
- `product` — List products in the catalog
- `product_feature` — List features for a product
- `implement_base` — SDK integration (prerequisite)

## Related Resources

- `monaiq://domain/model` — Entity relationships, feature type hierarchy
- `monaiq://domain/namespaces` — Authoritative type-to-namespace mappings
- `monaiq://sdk/dotnet/setup` — .NET SDK setup
- `monaiq://sdk/react/setup` — React SDK setup

</process>

<error-recovery>
## Error Recovery

| Failure Point | Symptom | Recovery Action |
|--------------|---------|----------------|
| Feature not found in catalog | `product_feature` list returns empty or missing feature | Verify the product code is correct. Route to `manage-catalog` to create missing features. |
| Wrong assertion type used | Runtime type mismatch error | Check the feature's `Kind` field — use `ServiceAccessLicenseAssertion` for Access features, `RateLimitLicenseAssertion` for RateLimit features. |
| Namespace import errors | Build fails with missing type references | Fetch `monaiq://domain/namespaces` and verify all `using`/`import` statements match the authoritative reference. |
| AssertLicense returns unexpected result | Feature check returns denied when it should be allowed | Verify the feature is assigned to the user's offering with the correct value (Allowed, not Denied). Check via `feature_offering` tool. |
| Consumption recording fails | `RecordConsumption` throws error for RateLimit features | Verify the feature type is RateLimit (not Access). Check that the consumption amount is a positive number. |

Feature gate code is additive — failed implementations can be corrected by editing the source file. No rollback needed.
</error-recovery>

<success_criteria>
- Correct assertion class used for each feature type (ServiceAccessLicenseAssertion or RateLimitLicenseAssertion)
- Feature checks return `IsAllowed = true` for valid licenses with the feature granted
- Application handles the denied case gracefully
- No namespace errors — all imports match `monaiq://domain/namespaces` reference
- Project builds without errors
</success_criteria>
