# ATDD Pipeline — Claude Code Skills

A test-first (red → green → refactor) development pipeline built as a set of
[Claude Code skills](https://docs.claude.com/claude-code). Every step is its
own SKILL.md, each one reads/writes a specific artifact file, and no single
step writes implementation or test code itself — authoring is always
delegated to a sub-agent (via the Agent tool) so the orchestrating session
never touches production files directly.

## The chain

```
atdd            requirement clarification → atdd.md
  ↓
threat-model    (conditional) abuse cases → AC-S<n> appended to atdd.md
  ↓
plan            read-only codebase exploration → plan.md
  ↓
test-copilot    failing tests written (red step, via sub-agent)
  ↓
code-copilot    implementation written to pass those tests (green step, via sub-agent)
  ↓
verify          build/lint/test/security gates → verify_report.md
  ↓
refactor        (conditional) structure cleanup while green
  ↓
red-team        independent review → red_team.json
  ↓
agentic-judge   (conditional) scores an LLM agent own tool-calling trajectory
  ↓
commit          only on explicit user request — never automatic
  ↓
postmortem      (out of the main chain) aggregates recurring findings across tasks
```

See skills/pipeline/SKILL.md for the full orchestration reference, including
which artifact each step reads/writes and which support skills apply to
which phase.
## What is in this repo

### Core pipeline

| Skill | Role |
|---|---|
| pipeline | Orchestration reference — names the chain, does not do work itself |
| atdd | Adaptive clarification questions → atdd.md (requirements + behaviour contract) |
| threat-model | Turns acceptance criteria into abuse cases before code is written |
| plan | Read-only file-impact plan (plan.md) before any authoring call |
| test-copilot | Dispatches a sub-agent to write failing tests (red step) |
| code-copilot | Dispatches a sub-agent to write the implementation (green step) |
| verify | Runs real quality gates (build/lint/test/security), reports PASS/FAIL |
| refactor | Behaviour-preserving cleanup once tests are green |
| red-team | Independent pre-commit review — security/correctness/architecture |
| agentic-judge | Scores an agent own tool-calling trajectory against a rubric |
| commit | User-approved-only commit step, Conventional Commit messages |
| postmortem | Aggregates recurring findings into checklist items for the next atdd.md |
| caveman, ponytail | Always-on communication/complexity discipline |
| fast-track | Skips the whole chain for trivial changes (typo, color tweak) |
| handoff | Writes a handoff doc when a session ends mid-task |
| wayfinder | Optional multi-session task breakdown for work too big for one session |

### Security and quality gates

| Skill | Role |
|---|---|
| security-scan | Deterministic security gate (secrets/SAST/dependency CVEs) |
| authz-test | Attacker-perspective authorization test matrix (IDOR, tenant leakage) |
| dynamic-pentest | Live agentic pentest — real payloads against a running target, not just code review |
| supabase-check | Verifies DB migrations actually apply and real requests work |
| trivy-scan | Dependency/secret/IaC/license scanning reference |

### Frontend and UI

| Skill | Role |
|---|---|
| frontend-pipeline | Extended UI/UX branch of the chain (Discover → Audit → Design → Implement → Validate) |
| frontend-audit | Playwright/Lighthouse/ZAP reference for live-URL UX/perf/security review |
| interface-review | Cross-category review: UI, typography, layout, color, writing, accessibility |
| better-accessibility | Accessibility standards and best-practice review |
| better-colors | Color systems — palettes, semantic tokens, contrast checks |
| better-layout | Grouping, alignment, reading order, progressive disclosure |
| better-typography | Type scale, spacing, variable fonts, wrapping, truncation |
| better-ui | UI polish — border radius, optical alignment, depth, hit areas |
| better-interface | Runs all better-* skills together as one combined review |
| theme-factory | Applies or generates a visual theme (colors/fonts) for artifacts/decks/docs |
| vision-test | Screenshot-based UI verification without spending vision tokens on the main session |
| webapp-testing | Playwright toolkit for driving/testing a local web app |

### Task-tracker integrations (optional)

| Skill | Role |
|---|---|
| jira-sync | Reads a Jira issue, creates/updates the matching Saga task — only skill allowed to talk to Jira/Saga |
| saga | Full Saga MCP reference plus a /saga <id> single-command pipeline entry point |

### Reference and meta

| Skill | Role |
|---|---|
| best-skill | Figures out a script/tool/workflow real usage by actually running it, no clarifying questions |
| lmstudio-test-review | Checklist a local LM Studio model uses when reviewing tests/code |

### Example domain-specific skills

These ship as worked examples of wrapping an external MCP/workflow into a
pipeline-compatible skill. They reference a specific personal setup (a job
search flow, a particular VPS) — copy and adapt them rather than using them
as-is; sensitive details (server addresses, personal email) have been
replaced with placeholders.

| Skill | Role |
|---|---|
| ai-job-search-outreach | End-to-end orchestration reference for a cold-outreach job search flow |
| indeed-jobs | Reference for using a connected Indeed MCP (search/details/company/resume) |
| job-search-calendar-gmail | Rules for using Calendar (auto) vs Gmail (always confirm before sending) in a job search flow |
| manus-notion-bridge | Hands a research task to Manus and reads the result back via a Notion page (not the Manus UI) |
| project-vault-cv-tailoring | Tailors a CV from a personal project-notes vault for a specific job posting |
| vps-deploy | Deploy-to-VPS runbook capturing known pitfalls (SSH access, env vars, PM2 restarts) |

## Install

Claude Code loads skills from .claude/skills/<name>/SKILL.md — either at
the user level (~/.claude/skills/, available in every project) or the
project level (<repo>/.claude/skills/, checked into that repo).

```bash
# user-level (available everywhere)
git clone https://github.com/<you>/atdd-pipeline-skills.git
cp -r atdd-pipeline-skills/skills/* ~/.claude/skills/

# or project-level (checked into one repo)
cp -r atdd-pipeline-skills/skills/* <your-repo>/.claude/skills/
```

Claude Code picks up new skills on the next session — no restart command
needed beyond starting a fresh claude session in that directory.

## Requirements

- Claude Code with the Agent/Skill tools available (current CLI).
- Sub-agent model access — test-copilot/code-copilot/plan open questions
  dispatch to a cheap model (Haiku) or a stronger one at low reasoning
  effort (Sonnet, reasoning_effort: low) depending on the step; any Claude
  model your account has access to works, adjust the model field in each
  SKILL.md Agent({...}) call to taste.
- A test runner and linter for your stack — verify shells out to whatever
  your project already uses (npm/pytest/etc.); it does not assume a
  specific one.
- Optional: a Supabase project (supabase-check), a Playwright or
  Lighthouse or ZAP setup (frontend-audit, webapp-testing), Trivy
  (trivy-scan), a Jira/Saga account (jira-sync, saga) — each of these
  skills is only invoked when the task actually touches that surface.

## Conventions worth knowing before you use this

- Git is never run by a sub-agent. Every dispatch prompt for
  test-copilot/code-copilot explicitly forbids add/commit/checkout/
  reset/restore/stash — a sub-agent that ran an unapproved commit and
  deleted uncommitted test files is why this rule exists (see
  skills/code-copilot/SKILL.md).
- commit only runs on explicit request, never as part of a full pipeline
  run — even a full-pipeline run stops after red-team and waits for an
  explicit commit or push.
- The behaviour-contract table in atdd.md is mandatory, not optional — it
  forces an explicit answer for partial success and nothing happened with
  no error either, the two failure modes most likely to hide a silent bug.
- plan is read-only. It never writes implementation or test code — only
  plan.md.
- The domain-specific example skills are personal-setup references, not
  drop-in tools — sensitive values (VPS address, email) are placeholders;
  swap in your own before using them.

## License

MIT — see LICENSE.
