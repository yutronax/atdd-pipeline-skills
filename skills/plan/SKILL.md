---
name: plan
description: Between atdd and test-copilot (test-first pipeline: plan → test-copilot → code-copilot) — reads atdd.md and the real codebase to produce a concrete file-change plan (files to modify, new files, dependencies, migrations, risks) before any authoring call is made. Read-only (Glob/Grep/Read only), never writes implementation or test code. Reduces rework by giving test-copilot/code-copilot a sharper extra_instructions target instead of guessing scope from atdd.md alone. Runs a mandatory frontend-pipeline trigger check (step 0) before its own discovery — a UI-touching task hands off to `frontend-pipeline` first instead of silently skipping it.
---

# Plan — file-impact planning, no authoring

## Kullanılacak destek skiller
- `caveman`, `ponytail` (full) — her zaman aktif.
- `superpowers:writing-plans` — plan.md'yi bu şablona göre yaz.
- `wayfinder` (opsiyonel) — iş tek session'a sığmayacak kadar büyükse plan.md yerine/yanında.

## Why this exists
Added 2026-07-30 to the pipeline (between `atdd` and `code-copilot`). Before
this, `code-copilot` derived its file list purely from atdd.md's own guesses
plus a cheap Codex classification call — often missing real dependencies
(a shared helper, a config file, a migration) until after an expensive
authoring call already ran. This skill does that discovery up front, for
free (no Codex call), using Claude's own read tools against the real repo.

## Precondition
`obss_project/artifacts/<task-slug>/atdd.md` must exist (from `atdd`,
optionally enriched by `jira-sync`). If missing, point at `atdd` — don't plan
against a task that was never clarified.

## Steps

0. **Frontend-pipeline tetikleyici kontrolü — zorunlu, atlanamaz (2026-09-12).**

   32 tamamlanmış görev geriye dönük tarandığında `frontend-pipeline`'ın
   **0/32** gerçekten çalıştığı bulundu (hiçbir `plan.md`'de "Discover"/
   "Audit bulgu"/"Design Direction" izi yok) — hatta `Görsel/UI kriteri`
   dolu olan görevlerde bile dedike `vision-test` skill'i atlanıp yerine
   ad-hoc Playwright ekran görüntüsü + doğrudan okuma kullanılmış. Sebep
   `threat-model`/`postmortem` ile aynı: tetikleme tamamen orkestratörün
   hatırlamasına bırakılmıştı. Bu adım o boşluğu kapatır — `plan`'ın
   kendisi asıl keşfe (adım 1-4) başlamadan önce bunu sormak zorunda.

   atdd.md'yi oku, `frontend-pipeline`'ın kendi tetikleyicilerini kontrol
   et: "Etkilenen Dosyalar" `.tsx`/`.jsx`/`.vue`/`.css`/`.html`/sayfa
   bileşeni içeriyor mu, VEYA "Benchmark / Başarı Ölçütü"ndeki "Görsel/UI
   kriteri" dolu mu.

   - **Tetikleyici varsa:** bu `plan` skill'ini DURDUR, `frontend-pipeline`
     skill'ini şimdi çağır (`Skill` tool). O kendi Discover→Audit→Design
     Direction adımlarını yürütüp çıktılarını `plan.md`'ye girdi olarak
     verecek (bkz. `frontend-pipeline` adım 4) — bu skill'in kendi
     1-4. adımları o zincirin PARÇASI olarak çalışır, ayrıca tekrar
     keşfedilmez.
   - **Tetikleyici yoksa:** `plan.md`'nin başına tek satır not düş:
     "Frontend-pipeline: tetikleyici yok (backend-only görev)." ve normal
     1. adıma geç.

   Bu kontrolü hiç yapmadan (sessizce) 1. adıma geçmek KABUL EDİLMEZ.

1. **Read atdd.md fully** — frontmatter (`affected_modules`, `priority`,
   `test_strategy`) and body (Acceptance Criteria, Kapsam Dışı, Rollback
   Beklentisi, Risks/Assumptions/Unknowns).
2. **Explore the real codebase** for each `affected_modules` entry and
   anything the Acceptance Criteria imply touching — `Glob`/`Grep`/`Read`
   only. Look for: existing patterns to follow (naming, error handling,
   response shapes), files that will need a matching change (e.g. a route
   file if a new endpoint is added), and anything the ATDD's "Etkilenen
   Dosyalar" section missed.

   **ZORUNLU kapsam sınırı:** `Glob`/`Grep` çağrılarını HER ZAMAN gerçek
   proje klasörüyle sınırla (`path` parametresi) — asla kök dizinden veya
   parametresiz geniş kapsamda arama yapma. Bu ortamlarda git reposunun
   kökü proje klasöründen DAHA GENİŞ olabilir (örn. ev dizini) ve devasa
   bir geçmiş/ilgisiz dosya kümesi taşıyabilir — sınırsız arama RAM'i
   tüketip yanıt vermeyebilir (bkz. atdd.md'nin "Proje Ortamı Kısıtı"
   bölümü, doluysa). atdd.md bu konuda "doğrulanmadı" diyorsa, ilk arama
   çağrısından önce hızlıca gerçek proje kökünü doğrula (`git rev-parse
   --show-toplevel` proje klasöründen ÇOK daha üstteyse dikkat).
3. **Check for migrations.** If the change implies a schema/data change
   (new table, new column, new required field), note it explicitly — this
   pipeline's projects generally forbid silent schema changes (see the
   target repo's own CLAUDE.md/ADRs for the actual rule, e.g. Group-9's
   ADR 0001 on `profiles.role`).
4. **Write the plan** to `obss_project/artifacts/<task-slug>/plan.md`:

```markdown
# Plan — <task-slug>
_Reference: atdd.md_

## Files to Modify
| File | Why | Risk |
|------|-----|------|
| path/to/file.py | <reason tied to an AC> | low/medium/high |

## New Files
| File | Purpose |
|------|---------|
| path/to/new_file.py | ... |

## Dependencies
<existing modules/functions this change will call or must stay consistent with>

## Migration Required?
Yes/No — <if yes, what and why; if the project forbids ad-hoc migrations,
say so and flag it for the user instead of assuming>

## Risks
<carried over/refined from atdd.md's Risks section, plus anything found here>

## Open Questions
<anything discovered during exploration that atdd.md didn't cover — ask the
user before code-copilot runs, don't guess>
```

Eğer "Files to Modify"/"New Files" bir rendered web UI dosyası (`.html`,
`.tsx`, `.jsx`, sayfa bileşeni, CSS) içeriyorsa bunu not et — bu, `verify`
adımında gate 11'in (`vision-test`) N/A değil aktif çalışacağı anlamına
gelir, sonraki adımlar bunu unutmasın.

5. **Open Questions varsa DUR VE KULLANICIYA SOR — otomatik alt-ajana dispatch etme.**

   (2026-09-19 düzeltmesi: bu adım önceden Open Questions'ı sessizce bir
   Sonnet alt-ajanına dispatch edip kullanıcıya hiç sormadan devam
   ediyordu. Bu, `atdd`'nin Hard Stop kuralıyla tutarsızdı — atdd.md'de
   belirsizlik varsa kullanıcıya soruluyor, ama aynı belirsizlik plan
   aşamasında kod keşfinden çıkarsa sessizce bir alt-ajan kendi kendine
   karar veriyordu. Artık ikisi de aynı kurala tabi: gerçek belirsizlik →
   kullanıcıya sor, dur.)

   - **Open Questions BOŞSA:** hiçbir şeyi durdurma, doğrudan 6. adıma geç.
   - **Open Questions DOLUYSA:** kullanıcıya `plan.md`'nin yolunu ver ve
     Open Questions bölümünü birebir göster, ŞUNU SOR: *"plan.md'yi
     oluştururken şu açık sorular çıktı: <liste>. Lütfen yanıtlayın ya da
     'sen karar ver' deyin. Onaylıyorsanız/yanıtladıysanız 'devam' deyin,
     test-copilot'a geçelim."* Kullanıcı "sen karar ver" derse (veya
     benzer bir yetki verirse), o zaman Sonnet 5 low alt-ajanına dispatch
     edilir (aşağıdaki prompt ile) ve kararlar `(kullanıcı yetkisiyle
     Sonnet 5 low alt-ajanı tarafından yanıtlandı)` notuyla `plan.md`'nin
     "Kararlar" bölümüne yazılır. Kullanıcı açık cevap verirse doğrudan o
     cevap kullanılır, alt-ajana gerek kalmaz.

   Alt-ajana dispatch prompt'u (yalnızca kullanıcı yetki verdiyse):
   ```
   Aşağıdaki plan.md'deki açık sorulara YANIT ver.
   BU BİR ARAŞTIRMA GÖREVİ DEĞİL — SADECE şu iki dosyayı Read et:
   <atdd.md mutlak yolu>, <plan.md mutlak yolu>. Başka HİÇBİR tool
   (Grep/Glob/WebSearch/Bash vb.) kullanma, kod tabanını tekrar tarama —
   kod keşfi zaten ana ajan tarafından yapılıp plan.md'ye işlendi.
   Bu iki dosyadan SADECE verilen bağlamdan en makul kararı seç.
   Her biri için en makul kararı 1-2 cümle gerekçeyle seç. Sorular: <Open
   Questions bölümü, birebir>.
   ```

   ```
   Agent({
     description: "Answer plan.md open questions for <task-slug>",
     subagent_type: "general-purpose",
     model: "sonnet",
     reasoning_effort: "low",
     run_in_background: false,
     prompt: "<yukarıdaki prompt, birebir>",
   })
   ```
6. **Open Questions yoksa (veya kullanıcı 5. adımda çözdüyse) DURMADAN doğrudan devam et.** Kullanıcıya `plan.md`'nin hazır olduğunu ve sonraki adımın `test-copilot` olduğunu söyle (test-first — red step, NOT `code-copilot` directly; implementation only follows once failing tests exist), which should pass this plan's "Files to Modify"/"New Files" lists as its file scope instead of re-deriving them from atdd.md alone — sonra **kendin** `test-copilot` skill'ini çağır, tekrar onay bekleme (atdd'nin Hard Stop'unda alınan "devam" onayı bu adımı zaten kapsıyor). `code-copilot` runs after `test-copilot`, per the pipeline order in `pipeline/SKILL.md`.

## Rule
- Read-only. `Glob`, `Grep`, `Read`, and the `plan.md` report file only —
  never `Write`/`Edit` on implementation or test files, never a Codex call.
- Don't pad the plan with speculative future-proofing — only files an
  Acceptance Criteria or a real dependency actually requires.
