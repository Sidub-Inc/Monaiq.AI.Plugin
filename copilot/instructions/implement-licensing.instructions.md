---
name: implement-licensing
description: SDK integration workflow with context flow — installs packages, configures credentials, registers services, and passes SDK config downstream
auto-invoke:
  - "User wants to integrate licensing SDK into their application"
  - "User wants to add license validation to their project"
  - "User asks how to set up Monaiq licensing in code"
tags: [sdk, integration, licensing, setup, dotnet, react]
category: integration
allowed-tools: [register_or_login, profile, implement_base]
argument-hint: "language (dotnet|react)"
tier: 2
invoked-by: [getting-started]
---

<input-context>
Receives from manage-catalog (or direct invocation):
- catalogSpec (optional): { productCode, features: [{ key, type }], offerings: [{ code, classification }] } — if provided, skip catalog verification
- language (optional): "dotnet" | "react" — if provided from user intent, skip language detection

If invoked without upstream context, the skill detects current state and asks the user for missing information.
</input-context>

<output-context>
Provides to downstream skills (implement-feature, implement-purchase-flow):
- sdkConfig: { language: "dotnet" | "react", credentialSource: "configuration" | "user-managed", serviceOptionsConfigured: boolean, diRegistered: boolean }
- targetFeatures: [{ featureKey, featureType }] — features available for gating (from catalog or user input)

Chain: manage-catalog [catalogSpec] → implement-licensing [sdkConfig] → implement-feature
</output-context>

<state-detection>
Before executing, check what's already set up:
1. Check project files for existing SDK packages:
   - .NET: Look for `Sidub.Licensing.Client` in *.csproj files
   - React: Look for `@sidub-inc/licensing-client` in package.json
2. Check for existing configuration:
   - .NET: Look for `LicensingServiceOptions` in appsettings.json
   - React: Look for LicensingProvider in source files
3. Check for DI registration:
   - .NET: Look for `AddSidubLicensing` or `AddSidubPlatform` in Program.cs
   - React: Look for `<LicensingProvider>` wrapper in App component

Based on detected state:
- Nothing found → Full flow from Step 1
- Packages installed → Skip Step 2, start at credential configuration
- Packages + config found → Skip to DI registration
- Everything configured → Show summary, offer credential recovery utility
</state-detection>

<objective>
Guide an agent through integrating the Monaiq licensing SDK into a .NET or React application — from credential discovery through runtime license validation. Covers six areas: credential source determination, package installation, namespace reference, service options configuration, dependency injection registration, and runtime usage.
</objective>

<process>

## Prerequisites

- Establish a session using the `register_or_login` tool
- Retrieve credentials via the `profile` tool (step 2) — you need your license key (called `EncodedCredential` in Monaiq)
- Fetch the SDK setup guide for the target language from `monaiq://sdk/{language}/setup`
- Fetch the namespace reference from `monaiq://domain/namespaces`

## Step 1: Determine Credential Source

Before writing any code, determine how end-users will provide their license credentials.

**Core question:** Do individual users or tenants each provide their own separate license credentials?

| Answer | Credential Source | Typical Use Case |
|--------|------------------|------------------|
| No | Configuration-Based | Single organization license, one credential in config |
| Yes | User-Managed | Multi-tenant SaaS, marketplace apps, per-user subscriptions |

**Configuration-Based** — one license key (`EncodedCredential`) in app settings covers the entire application. The SDK's built-in config-based credential resolver (`ConfigurationLicensingContextProvider`) handles this automatically.

**User-Managed** — each user/tenant provides their own credential at runtime. You must implement a credential resolver (the `ILicensingContextProvider` interface) to resolve credentials from your storage (user profile, tenant settings, etc.). Consider the `implement_purchase_flow` tool for embedded in-app purchases that provision credentials automatically.

## Step 2: Install Packages

### .NET

```bash
dotnet add package Sidub.Licensing.Client
```

All `Sidub.Platform.*` dependencies resolve automatically as transitive references.

### React

```bash
npm install @sidub-inc/licensing-client
```

React bindings are included in the same package — no separate React package needed.

## Step 3: Namespace Reference

Fetch `monaiq://domain/namespaces` for the authoritative type-to-namespace mappings. Key namespaces:

| Purpose | .NET Namespace | React Import |
|---------|---------------|--------------|
| Context & credentials | `Sidub.Licensing.Context` | `@sidub-inc/licensing-client` |
| DI registration | `Sidub.Licensing.Client` | N/A (use `LicensingProvider`) |
| Service interfaces | `Sidub.Licensing.Services` | `@sidub-inc/licensing-client` |
| Feature types | `Sidub.Licensing.Features` | `@sidub-inc/licensing-client` |
| Platform core | `Sidub.Platform.Core` | N/A |

**Critical:** Use ONLY the namespaces from the reference. Do not guess or assume type locations.

## Step 4: Configure Service Options

### .NET

Add your licensing configuration (called `LicensingServiceOptions` in the SDK) to `appsettings.json`:

```json
{
  "LicensingServiceOptions": {
    "LicenseServiceUri": "https://api.monaiq.com/licensing",
    "ConsumptionServiceUri": "https://licensing-consumption.sidub.ca",
    "EncodedCredential": "SIDUB_LIC_..."
  }
}
```

Bind in `Program.cs`:

```csharp
builder.Services.Configure<LicensingServiceOptions>(
    builder.Configuration.GetSection("LicensingServiceOptions"));
```

For **User-Managed** credential source, omit the license key (`EncodedCredential`) from config — it will be resolved at runtime by your custom credential resolver (`ILicensingContextProvider`).

### React

Pass config to `LicensingProvider`:

```tsx
<LicensingProvider config={{
  licenseServiceUri: 'https://api.monaiq.com/licensing',
  consumptionServiceUri: 'https://licensing-consumption.sidub.ca',
  encodedCredential: process.env.REACT_APP_SIDUB_CREDENTIAL
}}>
  <YourApp />
</LicensingProvider>
```

Store credentials in environment variables — never hardcode in source.

## Step 5: Register Dependency Injection

### .NET

```csharp
using Sidub.Licensing.Client;
using Sidub.Licensing.Client.Extensions;
using Sidub.Platform.Core;
using Sidub.Platform.Core.Services;

// Register Monaiq licensing services (AddSidubLicensing — includes AddSidubPlatform, AddSidubCryptography)
builder.Services.AddSidubLicensing();

// Register license endpoints with the platform service registry
builder.Services.AddSidubPlatform(registry =>
{
    var options = builder.Configuration
        .GetSection("LicensingServiceOptions")
        .Get<LicensingServiceOptions>()!;

    registry.RegisterLicense(options);
});
```

For **User-Managed** credential source, register your custom credential resolver:

```csharp
builder.Services.AddSidubLicensing<TenantLicensingContextProvider>();
```

### React

No additional DI setup — `LicensingProvider` handles all registration. Access the client via `useLicensingContext()` in any child component.

## Step 6: Runtime Usage

### .NET

Inject `ILicensingService` and verify the license:

```csharp
using Sidub.Licensing.Services;

public class MyService
{
    private readonly ILicensingService _licensingService;

    public MyService(ILicensingService licensingService)
    {
        _licensingService = licensingService;
    }

    public async Task CheckLicenseAsync()
    {
        var authorization = await _licensingService.GetAuthorization();
        // authorization is non-null when the license is valid
    }
}
```

### React

```tsx
import { useLicensingContext } from '@sidub-inc/licensing-client';

function LicenseStatus() {
  const client = useLicensingContext();
  const [status, setStatus] = useState('checking...');

  useEffect(() => {
    client.getAuthorization()
      .then(auth => setStatus(auth ? 'Licensed' : 'Not licensed'))
      .catch(err => setStatus(`Error: ${err.message}`));
  }, [client]);

  return <div>License status: {status}</div>;
}
```

## Verification

After completing the integration:

1. Build the project — no compilation errors
2. Run the application and verify `GetAuthorization()` returns a non-null result
3. Check logs for any `LicensingConfigurationException` — this indicates incomplete credential fields

## Related Tools

- `implement_base` — Interactive step-by-step SDK integration (follows the same 6-area structure)
- `register_or_login` — Establish a session before integration
- `profile` — Retrieve credentials (step 2) and onboarding status

## Related Resources

- `monaiq://sdk/dotnet/setup` — .NET SDK setup guide
- `monaiq://sdk/react/setup` — React SDK setup guide
- `monaiq://domain/namespaces` — Authoritative namespace-to-type mappings

## Utility Workflows

When state detection shows SDK is already integrated, offer these options:

- **Credential recovery**: Call `profile` tool to re-retrieve your license key (EncodedCredential). Useful when credentials are lost or need refreshing.
- **Update service URIs**: Modify your licensing configuration (`LicensingServiceOptions`) to point to different environments (staging, production).
- **Switch credential source**: Migrate from configuration-based to user-managed credentials (or vice versa) — involves implementing/removing a credential resolver (`ILicensingContextProvider`).
- **Verify integration**: Run a quick health check — confirm packages are installed, config is present, DI is registered, and a test license assertion succeeds.

</process>

<success_criteria>
- Project builds without compilation errors after SDK integration
- `GetAuthorization()` returns a non-null result at runtime
- No `LicensingConfigurationException` in application logs
- Correct credential source pattern used (Configuration-Based or User-Managed)
- All namespaces match the `monaiq://domain/namespaces` reference
</success_criteria>
