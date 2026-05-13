# Response Patterns

Use these lightweight shapes for user-facing Monaiq output. Keep business meaning first and compact technical backing second.

## Status Summary

- **Found:** current profile/catalog/SDK/journal facts.
- **Means:** what those facts imply for the licensing journey.
- **Next:** one primary next action.

## Evidence Summary

- **Backend facts:** profile, product, feature, offering, license, or checkout facts read back from MCP tools.
- **Code facts:** files, frameworks, SDK/config state, existing feature or checkout paths.
- **Journal facts:** current skill, checklist gate, last checkpoint, blockers, prior decisions.
- **Assumptions:** labeled assumptions to confirm at the next checkpoint.

## Checkpoint Prompt

- **Change:** intended catalog/config/code/purchase action.
- **Impact:** user/business effect.
- **Entities/files:** affected products, features, offerings, files, or credential stores.
- **Risks:** secret exposure, migration, checkout, entitlement, or validation risk.
- **Validation:** how success will be proven.
- **Options:** present via the **Host-Native Ask Pattern** in `_shared/protocols.md`. Standard checkpoint options: `approve automatic implementation`, `show a reviewable plan first`, `revise scope`, `stop`.

## Completion Summary

- **Changed:** backend entities and files changed.
- **Proof:** read-backs, commands, and behavior states verified.
- **Unresolved:** risks, skipped checks, blockers, or open questions.
- **Next:** next recommended skill or no further action.

## Evidence Backing

Specialist workflow steps that present a recommendation, plan, or proposed change should lead with a business-readable summary of impact, then provide compact technical backing. Skills should reference this pattern by name instead of restating its contents.

Technical backing fields:

- **Codebase evidence:** files, frameworks, SDK/config state, existing feature/checkout paths.
- **Journal decisions:** prior checkpoints, current checklist gate, recent milestone activity, blockers.
- **Backend facts:** profile, catalog, offering, license, or checkout facts read back from MCP tools.
- **Route-packet evidence:** active platform, target project, prior route assumptions, recommended skill, recommendation rationale.
- **Labeled assumptions:** inferred decisions to confirm at the next existing checkpoint.
- **Confidence and missing evidence:** what is known, what is inferred, and what would change the recommendation.

Inferred decisions must be confirmed at the next existing checkpoint with labeled assumptions. Do not introduce a new checkpoint name solely to record evidence inference.