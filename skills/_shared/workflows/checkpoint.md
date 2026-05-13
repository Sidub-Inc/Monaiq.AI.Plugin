# Checkpoint Workflow

Use this workflow for every hard checkpoint. This shared source file is assembled into the multi-target Monaiq plugin output for Claude, GitHub Copilot / VS Code, and Codex, so the host-specific ask behavior must be explicit and portable.

1. **Prompt:** call `monaiq_journal save_checkpoint` with `checkpoint`, `summary`, `question`, intended change, user/business impact, affected entities/files/areas, secret policy, risks, validation plan, and safe context. This prompt call returns no file operations.
2. **Ask:** present the checkpoint using the **Host-Native Ask Pattern** in `_shared/protocols.md`. For checkpoint gates the standard options are `approve automatic implementation`, `show a reviewable plan first`, `revise scope`, and `stop`.

   **VS Code / GitHub Copilot** example invocation:
   ```
   vscode_askQuestions([{
     "header": "Monaiq — [CHECKPOINT-NAME]",
     "question": "[checkpoint question]",
     "options": [
       { "label": "Approve automatic implementation", "value": "approve", "recommended": true },
       { "label": "Show a reviewable plan first", "value": "plan" },
       { "label": "Revise scope", "value": "revise" },
       { "label": "Stop", "value": "stop" }
     ]
   }])
   ```
   For Claude / Claude Code, use `AskUserQuestion` with the same shape. For Codex / headless, present a numbered list and require a numeric reply. Do not continue until the user's answer is recorded in the follow-up `monaiq_journal save_checkpoint` call with the `result` payload.
3. **Approval + write:** after the user answers, call `monaiq_journal save_checkpoint` again with the same `checkpoint`, `summary`, `question`, and a `result` payload such as `{ "answer": "approved" }`.
4. **Apply:** apply returned `.monaiq/*` file operations locally after validating each path against the `.monaiq` allowlist.
5. **Verify:** verify the expected `.monaiq/CHECKPOINTS/*` file exists before claiming the checkpoint is recorded.

Use `checkpoint` in tool payloads. Do not use `checkpointName` except as route-packet display metadata.

Failure to complete prompt, ask, approval write, operation application, and local verification means the checkpoint has not been durably recorded. Do not continue into catalog mutation, credential/config writes, application behavior edits, purchase-flow changes, or validation remediation until the checkpoint is recorded or the user selects a stop/revise option.

Do not infer hard approval from casual chat such as "looks good", "i like it", "sure", or similar conversational agreement. Catalog mutation, code/config edits, credential writes, platform-scope expansion, checkout behavior, and validation remediation require an explicit host-native checkpoint answer and a durable checkpoint result.

Each checkpoint prompt must include: summary, question, intended change, user/business impact, affected entities/files/areas, assumptions, secret policy, risks, validation plan, and options. Default options are `approve automatic implementation`, `show a reviewable plan first`, `revise scope`, and `stop`.

Hard checkpoints include `CHECKPOINT-WORKFLOW-START`, `CHECKPOINT-PRE-CATALOG-MUTATION`, `CHECKPOINT-PRE-CREDENTIAL-WRITE`, `CHECKPOINT-PRE-BUSINESS-LOGIC-EDIT`, `CHECKPOINT-CHECKOUT-ARCHITECTURE`, `CHECKPOINT-PRE-CREDENTIAL-PERSISTENCE`, and `CHECKPOINT-PRE-APIKEY-EXPOSURE-RISK`.