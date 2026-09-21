---
name: jira-sync
description: Reads a Jira issue and creates/updates the matching Saga task — the ONLY skill in the pipeline allowed to talk to Jira or Saga. Returns issue summary + AC + saga_task_id for atdd (and later status updates for the tail end of the pipeline). KURAL: Tek sorumluluk prensibi gereği atdd\'den ayrı çalışır. — atdd stays tool-independent.
---

# Jira Sync — Jira + Saga integration only

## Why this exists
Split out of `atdd` on 2026-07-30 (feedback: `atdd` was doing three unrelated
jobs — clarifying requirements, syncing Jira/Saga, and orchestrating the
pipeline — and Saga integration was the most likely piece to break or change
independently). This skill's only job is talking to Jira and Saga. It knows
nothing about Acceptance Criteria wording, question counts, or benchmarks —
that's `atdd`'s job, using this skill's output as input.

## Fixed IDs (don't re-look these up)
- **Jira Cloud ID:** `1c97069a-da45-4ac5-916e-0d51b8cc6ab4`
  (`<your-domain>.atlassian.net`). Project: **KAN** (`obss-project_team`).
  Epics `KAN-4`..`KAN-11` (Epic 1-8) + `KAN-125` (optional/uncertain).
- **Saga project:** `project_id: 4` ("OBSS Bridge"), epic: `epic_id: 8`
  ("OBSS Bridge"). 

## When to call this skill
- `atdd` calls it when the user references a Jira ID ("KAN-14'ten devam",
  "kaldığımız yerden devam") or asks "nerede kaldık" — check Saga FIRST here,
  not conversation history.
- The tail end of the pipeline (after `verify`/`red-team`, or when `commit`
  finishes) calls it to update the Saga task's status — see "Status updates"
  below.

## Steps

### A. Lookup for a new or resumed task (called from `atdd`)
1. **Check Saga first** for an existing open task before doing anything else:
   `mcp__saga__task_list` (epic_id: 8, no status filter), or if a Jira ID is
   known, `mcp__saga__tracker_search`. This answers "nerede kaldık" —
   never scan chat history for this, Saga is the source of truth.
2. If a Jira ID was given and no Saga task exists yet for it, fetch the issue:
   `mcp__5187d3dc-7550-4116-b761-0b06f95398e5__getJiraIssue` with the fixed
   Cloud ID above.
3. **Create the Saga task** (before `atdd.md` itself is written):
   ```
   mcp__saga__task_create({
     epic_id: 8,
     title: "<JIRA-ID>: <short summary>",
     description: "<Jira epic/story full summary + will-be atdd.md path>",
     status: "todo",
     priority: "high" | "medium" | "low",   // mirror Jira's priority exactly
     tags: ["<JIRA-ID>", "epic-N", "atdd"]
   })
   ```
4. **If resuming** (Saga task already exists): `mcp__saga__task_update({id,
   status: "in_progress"})` instead of creating a new one.
5. Return to the caller (`atdd`): Jira issue summary, full AC text verbatim,
   priority, epic link, and the `saga_task_id`. `atdd` embeds this verbatim
   into `atdd.md`'s frontmatter/Jira section — it must not re-query Jira.

### B. Status updates (called after verify/red-team/commit)
Always ask before writing, via `AskUserQuestion`:
```
"<task> için Saga görevini (id: <saga_task_id>) 'review'/'done' olarak
güncelleyeyim mi?"
Seçenekler: "Evet, güncelle" / "Hayır, henüz değil"
```
If yes: `mcp__saga__task_update({id: <saga_task_id>, status: "review"|"done",
actual_hours: <if known>})`. Don't skip asking silently — "nerede kaldık"
answers depend on Saga staying current.

## Rule
- This is the only skill that calls Jira/Saga MCP tools in the pipeline.
  `atdd`, `plan`, `code-copilot`, `test-copilot`, `verify`, `red-team`, and
  `commit` all read the IDs/summary this skill already produced instead of
  querying Jira/Saga themselves.
- Never invent a Jira ID or Saga task_id — if lookup fails, say so and ask
  the user, don't guess.
