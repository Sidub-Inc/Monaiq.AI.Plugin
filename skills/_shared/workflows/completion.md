# Completion Workflow

Before a skill exits or hands off to another skill, record only meaningful completion state:

1. Coalesce changed paths, decisions, checklist progress, validation proof, unresolved risks, and downstream handoff data into the fewest journal actions the current `monaiq_journal` tool supports. Do not include file contents or secret values.
2. Call `monaiq_journal record_file_changes` only when workspace paths actually changed; include changed paths and intent only.
3. Call `monaiq_journal update_checklist_progress` only for gates owned by the current skill, and only after source skill, relevant MCP tool, canonical resources, and checkpoint/journal evidence are recorded.
4. Save `CHECKPOINT-SKILL-COMPLETE` only when the completion itself is consequential: files or backend state changed, validation status changed, a downstream skill needs durable proof, or the user-visible journey gate changed. Do not save it after read-only explanation or when `skill_completed` plus the state packet is enough.
5. Include a no-secret `proofOfDone` packet in `CHECKPOINT-SKILL-COMPLETE` when the checkpoint is used.
6. After every `monaiq_journal` call in this skill — `skill_started`, `update_checklist_progress`, `save_checkpoint`, `skill_completed`, and any other journal call that returns `fileOperations` — apply them immediately using the **File Operation Application Protocol** in `_shared/protocols.md`. Do not defer or coalesce file operation application. Verify expected `.monaiq` files after each application.
7. Call `monaiq_journal skill_completed` once per user-visible skill outcome, not after every internal tool step.
8. Recommend the next specialist skill or stop with unresolved risks.
9. **Next Step Signal (mandatory):** emit as the final line of every completion output:

   > **Next:** Invoke `[skill-name]` — [one sentence describing what it does for the user's journey].

   If no next skill applies because unresolved risks block continuation, emit instead:

   > **Next:** Resolve [blocker description] before continuing.

   Every skill completion path must produce exactly one Next Step Signal. Omitting it is not permitted.

Routine completion records should be sparse. A typical successful implementation skill has startup/resume, one or more hard checkpoint result writes, validation failure writes only if needed, and one completion/progress batch. Avoid separate back-to-back journal calls for file changes, checklist progress, completion checkpoint, and lifecycle when the same milestone can be represented more compactly without losing required evidence.

## proofOfDone Schema

```json
{
  "backendReadBack": ["backend facts verified through MCP tools"],
  "filesChanged": ["workspace-relative paths affected"],
  "commandsRun": ["build/test/check commands and results"],
  "behaviorVerified": ["allow/denied/expired/over-limit/checkout states checked"],
  "unverifiedRisks": ["what remains untested or deferred"],
  "nextRecommendedSkill": "skill name or null"
}
```

`manage-catalog` proof must include read-back verification. Code skills must include build/test commands and behavior states checked.