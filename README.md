# ATDD Pipeline — Claude Code Skills

A test-first (red → green → refactor) development pipeline built as a set of
[Claude Code skills](https://docs.claude.com/claude-code). Every step is its
own `SKILL.md`, each one reads/writes a specific artifact file, and no single
step writes implementation or test code itself — authoring is always
delegated to a sub-agent (via the `Agent` tool) so the orchestrating session
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
agentic-judge   (conditional) scores an LLM agent's own tool-calling trajectory
  ↓
commit          only on explicit user request — never automatic
  ↓
postmortem      (out of the main chain) aggregates recurring findings across tasks
```

See [`skills/pipeline/SKILL.md`](skills/pipeline/SKILL.md) for the full
orchestration reference, including which artifact each step reads/writes and
which support skills apply to which phase.

## What's in this repo

| Skill | Role |
|---|---|
| `pipeline` | Orchestration reference — names the chain, doesn't do work itself |
| `atdd` | Adaptive clarification questions → `atdd.md` (requirements + behaviour contract) |
| `threat-model` | Turns acceptance criteria into abuse cases before code is written |
| `plan` | Read-only file-impact plan (`plan.md`) before any authoring call |
| `test-copilot` | Dispatches a sub-agent to write failing tests (red step) |
| `code-copilot` | Dispatches a sub-agent to write the implementation (green step) |
| `verify` | Runs real quality gates (build/lint/test/security), reports PASS/FAIL |
| `security-scan` | Deterministic security gate (secrets/SAST/dependency CVEs) |
| `authz-test` | Attacker-perspective authorization test matrix (IDOR, tenant leakage) |
| `supabase-check` | Verifies DB migrations actually apply and real requests work |
| `trivy-scan` | Dependency/secret/IaC/license scanning reference |
| `refactor` | Behaviour-preserving cleanup once tests are green |
| `red-team` | Independent pre-commit review — security/correctness/architecture |
| `agentic-judge` | Scores an agent's own tool-calling trajectory against a rubric |
| `vision-test` | Screenshot-based UI verification without spending vision tokens on the main session |
| `frontend-audit` | Playwright/Lighthouse/ZAP reference for live-URL UX/perf/security review |
| `commit` | User-approved-only commit step, Conventional Commit messages |
| `postmortem` | Aggregates recurring findings into checklist items for the next `atdd.md` |
| `caveman`, `ponytail` | Always-on communication/complexity discipline |
| `fast-track` | Skips the whole chain for trivial changes (typo, color tweak) |
| `handoff` | Writes a handoff doc when a session ends mid-task |
| `wayfinder` | Optional multi-session task breakdown for work too big for one session |

**Not included:** skills that were tightly coupled to a specific personal
setup (a particular task tracker's MCP, a specific CV/job-search workflow, a
particular VPS). If you use a task tracker (Jira, Linear, Saga, etc.), the
`atdd`/`pipeline` skills reference an optional `jira-sync`/`saga`-style
integration step — write your own thin wrapper skill for your tracker's MCP
and slot it in before `atdd`; nothing else in the chain depends on it.

## Install

Claude Code loads skills from `.claude/skills/<name>/SKILL.md` — either at
the user level (`~/.claude/skills/`, available in every project) or the
project level (`<repo>/.claude/skills/`, checked into that repo).

```bash
# user-level (available everywhere)
git clone https://github.com/<you>/atdd-pipeline-skills.git
cp -r atdd-pipeline-skills/skills/* ~/.claude/skills/

# or project-level (checked into one repo)
cp -r atdd-pipeline-skills/skills/* <your-repo>/.claude/skills/
```

Claude Code picks up new skills on the next session — no restart command
needed beyond starting a fresh `claude` session in that directory.

## Requirements

- **Claude Code** with the `Agent`/`Skill` tools available (current CLI).
- **Sub-agent model access** — `test-copilot`/`code-copilot`/`plan`'s open
  questions dispatch to a cheap model (Haiku) or a stronger one at low
  reasoning effort (Sonnet, `reasoning_effort: "low"`) depending on the
  step; any Claude model your account has access to works, adjust the
  `model:` field in each `SKILL.md`'s `Agent({...})` call to taste.
- **A test runner and linter for your stack** — `verify` shells out to
  whatever your project already uses (npm/pytest/etc.); it doesn't assume
  a specific one.
- **Optional:** a Supabase project (`supabase-check`), a Playwright/
  Lighthouse/ZAP setup (`frontend-audit`), Trivy (`trivy-scan`) — each of
  these skills is only invoked when the task actually touches that surface.

## Conventions worth knowing before you use this

- **Git is never run by a sub-agent.** Every dispatch prompt for
  `test-copilot`/`code-copilot` explicitly forbids `add`/`commit`/`checkout`/
  `reset`/`restore`/`stash` — a sub-agent that ran an unapproved commit and
  deleted uncommitted test files is why this rule exists (see
  [`skills/code-copilot/SKILL.md`](skills/code-copilot/SKILL.md)).
- **`commit` only runs on explicit request**, never as part of "run the
  whole pipeline" — even a full-pipeline run stops after `red-team` and
  waits for an explicit "commit"/"push".
- **The behaviour-contract table in `atdd.md` is mandatory**, not optional —
  it forces an explicit answer for "partial success" and "nothing happened,
  no error either", the two failure modes most likely to hide a silent bug.
- **`plan` is read-only.** It never writes implementation or test code —
  only `plan.md`.

## License

MIT — see [LICENSE](LICENSE).
