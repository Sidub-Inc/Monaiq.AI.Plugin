---
name: troubleshoot-integration
description: Fix SDK integration issues — diagnose setup errors, authentication failures, license validation problems, and usage tracking bugs
auto-invoke:
  - "User is having trouble integrating the Monaiq licensing SDK and sees errors or unexpected behavior"
  - "User encounters an error with licensing setup, authentication, or validation — not asking how to set up from scratch"
  - "User asks for help debugging a licensing integration issue — something was working or partially working and now isn't"
  - "User reports a specific error message, exception, or HTTP status code related to licensing"
tags: [integration, troubleshooting, diagnostics, errors]
category: integration
allowed-tools: [register_or_login, profile, product, offering, mcp__plugin_monaiq_monaiq__register_or_login, mcp__plugin_monaiq_monaiq__profile, mcp__plugin_monaiq_monaiq__product, mcp__plugin_monaiq_monaiq__offering]
argument-hint: "errorCategory (setup|auth|validation|consumption)"
tier: 1
invoked-by: [user]
---

<output-context>
Provides to requesting context:
- diagnosis: { category: "setup" | "auth" | "validation" | "consumption", symptom, resolution, resolved: boolean }

This is a support entry point — users come here directly when something is broken.
</output-context>

<objective>
Diagnose and resolve Monaiq SDK integration issues. Self-diagnose first from available context (error messages, code, configuration), then fall back to interactive diagnostic questions if the category or resolution can't be determined autonomously. Reference `monaiq://troubleshooting` as the diagnostic knowledge source.
</objective>

<process>

## Prerequisites

- Fetch `monaiq://troubleshooting` for structured decision trees across all 4 error categories
- Fetch `monaiq://sdk/{language}/setup` for the user's SDK language if known

## Step 1: Self-Diagnose

Attempt to determine the error category and specific symptom automatically from:
- Error messages in the conversation (stack traces, HTTP status codes, exception types)
- Code context (if the user shared code snippets or files)
- Configuration context (if appsettings.json, .env, or package.json is visible)

Map to one of the 4 categories:

| Category | Signals |
|----------|---------|
| **Installation and setup issues** | Package install errors, NuGet/npm failures, missing config sections, DI registration exceptions |
| **Connection and credential problems** | 401/403 responses, "API key" mentions, connection refused, credential errors |
| **License check failures** | License not found, expired, revoked, wrong product, unexpected entitlement values |
| **Usage tracking and purchase flow issues** | Consumption reporting failures, checkout session errors, redirect issues |

If a specific symptom is identified, proceed directly to the matching decision tree from the troubleshooting resource and follow its diagnostic steps to resolution.

## Step 2: Interactive Fallback

If the category cannot be determined automatically:
- Ask what the user is trying to do (install SDK, authenticate, validate license, report consumption, complete checkout)
- Ask for the specific error message or observed behavior
- Map responses to the appropriate error category and symptom

## Step 3: Apply Decision Tree

Using the matching decision tree from `monaiq://troubleshooting`:
1. Present the diagnostic steps from the resource
2. Work through each step with the user
3. Identify the root cause
4. Present the resolution

Reference the troubleshooting resource content — do not duplicate it. Fetch the resource and use its decision trees directly.

## Step 4: Verify Resolution

After applying the resolution, confirm the issue is resolved:
- Ask the user to retry the operation
- If still failing, re-evaluate: check for a different root cause or escalate to a different error category

</process>

<success_criteria>
- `monaiq://troubleshooting` was fetched and used as the diagnostic knowledge base
- Self-diagnosis was attempted first from available error context
- Interactive fallback was used only if self-diagnosis was insufficient
- All 4 error categories are coverable (setup, auth, validation, consumption/checkout)
- Resolution was applied and verified
</success_criteria>
