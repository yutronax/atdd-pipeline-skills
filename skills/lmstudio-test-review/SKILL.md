---
name: lmstudio-test-review
description: Checklist LM Studio (local model) uses when reviewing tests/code for a task's "test" category -- ATDD acceptance criteria grounding, test pyramid ratio (unit/integration/e2e), and a fixed code-smell checklist (God function, magic number, deep nesting, long parameter list). Loaded by review_code()/review_tests() calls to LM Studio -- not meant to be read standalone outside that flow.
---

# LM Studio Test/Quality Review Checklist -- FULL REFERENCE

This is the fixed rubric sent to LM Studio (local model, `qwen2.5-coder-7b-instruct`
via `C:\obss_bridge\ask_codex.py`) whenever the `test` category (or the
step 6 final review) needs to judge test coverage/quality, not just "does
it run." Keep this file itself short and mechanical -- it gets inlined
into the LM Studio prompt, so verbosity here directly costs prompt tokens
on a small local model.

## 1. Ground every judgment in the ATDD task JSON, not vibes

Before judging anything, the reviewer must have the task's `user_scenarios`
and `possible_tests` (from the step-2 ATDD JSON) in front of it. A test
suite is "correct" ONLY relative to those -- not relative to generic best
practice. Checklist:
- Does at least one test exist per `possible_tests` entry?
- Does at least one test (unit or integration) verify each `user_scenarios`
  entry actually holds?
- Flag any `user_scenarios` entry with ZERO covering test as a gap, named
  explicitly (don't just say "coverage could be better").

## 2. Test pyramid -- ratio + min/max per layer

Classic pyramid, applied as a review heuristic (not a hard gate -- flag
violations, don't block on them):

| Layer | Target share of total tests | Min | Max | What belongs here |
|---|---|---|---|---|
| **Unit** | ~70% | At least 1 per pure function / business-logic branch with real decision logic (conditionals, calculations, validation) | No fixed max -- more unit tests is fine as long as each still tests ONE thing | Pure logic, no I/O, no network, no real DB/filesystem, fast (ms) |
| **Integration** | ~20% | At least 1 per seam where two real components meet (module+DB, API+auth layer, two internal modules) that unit tests can't safely fake | Roughly ≤ half the unit test count -- if integration tests outnumber unit tests, logic is probably not decomposed enough | Real DB/filesystem/subprocess involved, but not the full running app/UI |
| **E2E** | ~10%, often less | At least 1 per CRITICAL user journey (the primary happy path from `user_scenarios`) | Small and deliberate -- 1-3 per feature is typical; more than that is usually redundant with integration tests and just adds flakiness/slowness | Full app/UI/API surface, closest to what the real user does |

Review output should flag: (a) zero tests at a layer that clearly needs
one (e.g. a security-relevant `user_scenarios` entry with no integration
test), (b) an inverted pyramid (more e2e than unit, or heavy reliance on
e2e for things a unit test could cover faster), (c) a single test file
mixing layers without saying so.

## 3. Fixed code-smell checklist

Apply to the files under review, cite the exact function/line pattern
when flagging (not a vague "this could be cleaner"):

- **God function** (tek fonksiyon çok fazla iş yapıyor): symptom is "no
  one fully understands this function" / it does several unrelated things
  (e.g. validates input AND queries DB AND formats output AND sends
  email in one function body). Fix: split into smaller single-purpose
  functions, one per responsibility.
- **Magic number** (sihirli sayı): an unexplained literal constant whose
  meaning isn't obvious from context, e.g. `if (total > 1000) total = total * 0.9`.
  Fix: name it, e.g. `INDIRIM_ESIGI = 1000`, `INDIRIM_ORANI = 0.9`.
- **Deep nesting** (derin iç içe geçme): stacked `if`/`for` blocks (3+
  levels) that make control flow hard to follow. Fix: early return /
  guard clauses, or extract the inner block into its own function.
- **Long parameter list** (uzun parametre listesi): a function signature
  like `f(o, u, cfg, db, mailer, logger, flags)` where the caller has to
  remember argument order/meaning. Fix: group related parameters into
  one object/struct/dataclass and pass that instead.

## Output JSON schema (what the reviewer must return)

```json
{
  "atdd_coverage": {
    "uncovered_user_scenarios": ["..."],
    "uncovered_possible_tests": ["..."]
  },
  "test_pyramid": {
    "unit_count": 0, "integration_count": 0, "e2e_count": 0,
    "issues": ["e.g. 'no unit tests for calculateDiscount(), only one e2e test covers it'"]
  },
  "code_smells": [
    {"type": "god_function|magic_number|deep_nesting|long_parameter_list", "location": "file:function/line", "detail": "...", "fix": "..."}
  ],
  "overall_impression": "..."
}
```

## How this is invoked

When the task involves test/quality judgment beyond a plain
syntax check, inline this checklist's sections 1-3 into the LM Studio
prompt (via `ask_codex.review_code()`/a `review_tests()` variant) along
with the ATDD task's `user_scenarios`/`possible_tests` and the files under
review, and request the schema in section 4. Do not paraphrase the
rubric loosely -- send the fixed definitions above so the local model
applies the SAME bar every time instead of improvising.
