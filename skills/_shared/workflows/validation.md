# Validation Workflow

Validation is evidence, not transcript confidence.

1. Select the smallest command set that proves the changed behavior.
2. Run build/test/lint or manual smoke commands appropriate to the host app.
3. Verify positive and negative behavior where applicable: allowed, denied, expired, over-limit, checkout success, checkout recovery, and unlicensed states.
4. On failure, call `monaiq_journal record_validation_failure` before remediation with command, observed result, likely cause, blocked next action, retry options, and checkpoint requirement.
5. Stop before further edits when remediation changes business logic, credentials, catalog state, or purchase behavior; use the checkpoint workflow.
6. Record unverified risks in `proofOfDone` rather than claiming success.