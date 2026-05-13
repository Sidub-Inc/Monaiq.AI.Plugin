# Gate Prompt Patterns

Use short, host-specific prompts. Monaiq guidance is distributed to Claude, GitHub Copilot / VS Code, and Codex runtimes, so checkpoint behavior must be clear even though each host exposes different interaction tools.

Avoid open-ended checkpoint questions unless the decision is genuinely undefined.

Host mapping:
- GitHub Copilot / VS Code: use `vscode_askQuestions` with concise headers, explicit options, and freeform enabled only when useful.
- Claude: use the available blocking approval, confirmation, or question mechanism for the Claude runtime. If no structured mechanism is exposed, ask one blocking question in chat, present explicit options, and wait for an explicit option/value answer before continuing.
- Codex: use the available blocking approval, confirmation, or question mechanism for the Codex runtime. If no structured mechanism is exposed, ask one blocking question in chat, present explicit options, and wait for an explicit option/value answer before continuing.

Do not name a non-existent host tool. When a target runtime lacks a structured ask tool, the blocking chat fallback is the compatibility path, but it still requires an explicit answer and durable checkpoint recording.

Fallback answers still must be written through `monaiq_journal save_checkpoint` before consequential work continues. Never treat casual phrases such as "i like it" as approval for catalog, code, credential, platform, checkout, or remediation changes.

## Standard Approval

- Approve
- Revise
- Stop

## Workflow Control

- Continue
- Pause
- Escalate

## Binary Choice

- Yes
- No

## Strategy Choice

- Option A
- Option B
- Let Monaiq decide from evidence

## Platform Scope Choice

- Continue with the active platform only
- Add a secondary platform to scope
- Revise target project or platform
- Stop

Each prompt must include the intended change, user/business impact, affected entities/files, risk summary, and validation plan.