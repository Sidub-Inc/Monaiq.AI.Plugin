# Handoff Schemas

Persist handoff packets in `.monaiq/STATE.md` summaries or `.monaiq/JOURNAL.md` entries with no secrets. Downstream skills should resume from these packets instead of re-asking questions already answered by evidence.

## routePacket

- `scenario`
- `targetApp`
- `platform`
- `activePlatform`
- `targetProject`
- `outOfScopePlatforms`
- `profileState`
- `catalogState`
- `codeEvidenceSummary`
- `journalEvidenceSummary`
- `assumptions`
- `recommendedSkill`
- `recommendationRationale`
- `checkpointName`
- `authorizedBy`

`activePlatform`, `targetProject`, and `outOfScopePlatforms` are routing boundaries, not decoration. Persist them after workflow start, catalog mutation approval, SDK setup approval, secondary-platform approval/denial, and validation-failure checkpoints so downstream skills do not broaden scope silently.

## catalogSpec

- `productCode`
- `features[]`
- `offerings[]`
- `assignments[]`

## pricingPlan

- `tiers[]` with `name`, `classification`, `baseRate`, `currency`, `interval`, and `features[]`

## sdkConfig

- `platform`
- `activePlatform`
- `targetProject`
- `outOfScopePlatforms`
- `credentialSource`
- `serviceOptionsConfigured`
- `diRegistered`

## scopeBoundary

- `activePlatform`
- `targetProject`
- `approvedSecondaryPlatforms[]`
- `outOfScopePlatforms[]`
- `checkpointName`
- `authorizedBy`

Use `scopeBoundary` when a journey might touch more than one project or stack. Do not add React, Node, Console, or another target to a .NET/Blazor implementation unless the checkpoint result explicitly approves that expansion.

## featureImpl

- `implementedFeatures[]`
- `buildVerified`

## checkoutImpl

- `architecture`
- `offeringCodes[]`
- `checkoutRoute`
- `successHandler`

## validationProof

- `passed[]`
- `failed[]`
- `skipped[]`
- `unverified[]`