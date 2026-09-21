---
name: pipeline
description: Orchestration reference for the full TDD/ATDD pipeline (atdd → jira-sync → threat-model → plan → test-copilot → code-copilot → verify → refactor → red-team → dynamic-pentest → commit → postmortem — test-first, red→green→refactor). Doesn't do any work itself — it names which skill runs next and what each hands to the next. Use when the user asks to run the whole flow, or to figure out "what's the next step" for a task already in progress.
---

# Pipeline — orchestration reference only

## Why this exists
Split out of `atdd` on 2026-07-30 — `atdd` used to both build the ATDD draft
AND decide when to hand off to `code`/`test`/`red-team`. This skill is just
the map of the chain, so no single step-skill needs to know what comes after
it. Each step-skill still ends by naming the next one and stopping — this
skill doesn't auto-chain across turns either, it's a reference to consult,
not a background process.

## Zincire hiç girmeden: fast-track

Görev çok küçük/düşük riskliyse (yazım hatası, CSS renk değişimi, ufak linter
düzeltmesi) — **zincir hiç başlamaz.** `atdd`'nin kendi 0. adımı bunu kontrol
eder ve tetiklenirse doğrudan `fast-track` skill'ine devreder: o da kodu
yazar, `verify`'ın linter/format kısmını çalıştırır ve commit atar — ATDD,
`plan`, `test-copilot` hiç çağrılmaz. Bu yüzden `fast-track` aşağıdaki
zincir diyagramında YOK — zincirin bir adımı değil, zincirin *alternatifi*.
"Bu görev fast-track mi, tam zincir mi?" sorusunu her zaman `atdd`'nin 0.
adımı cevaplar, burada ayrıca karar verilmez.

## The chain

```
atdd  (+ jira-sync when a Jira ID is involved)
  ↓
threat-model   (conditional — abuse cases become AC-S<n> in atdd.md)
  ↓
plan
  ↓
test-copilot   (red — failing tests against atdd.md, no implementation yet)
  ↓
code-copilot   (green — implementation written to satisfy those tests)
  ↓
verify         (yeşil kapısı — testler geçmeli; security-scan burada gate 10)
  ↓
refactor       (koşullu — yeşilken yapı iyileştirmesi, davranış değişmez)
  ↓
red-team
  ↓
dynamic-pentest (conditional — only when atdd.md has real AC-S<n> criteria from
                threat-model, i.e. the task actually touches auth/payment/
                multi-tenant/file-upload; sits alongside red-team, doesn't
                replace it — red-team reasons about the diff, dynamic-pentest
                (a Claude subagent, no external CLI) tries to actually
                exploit it)
  ↓
agentic-judge  (conditional — only when atdd.md's AC targets an LLM agent/sub-agent's
                tool-calling trajectory or response quality; sits alongside red-team,
                doesn't replace it)
  ↓
commit   (only on explicit user request — never automatic)
  ↓
postmortem     (zincirin dışında — görevler biriktikçe kalıp çıkarır)
```
`refactor` koşulludur: testler yeşilse ve `code_diff.md`'de iyileştirmeye değer
bir şey varsa çalışır. Testler kırmızıyken **asla** çalışmaz — refactor'ün tek
güvencesi testlerdir. `verify` kendini "refactor gate" diye tanıtıyordu ama
sadece kontrol koşuyordu; asıl iyileştirme adımı budur.
**2026-09-12'den itibaren `verify`'ın kendisi bu kararı zorunlu hale getirdi**
(32 tamamlanmış görevde `refactor` 1/32 çalışmıştı — o tek sefer de kullanıcının
elle çağırmasıyla oldu) — testler yeşilse `verify` raporunu bitirmeden önce
somut bir refactor adayı olup olmadığını değerlendirip kullanıcıya bildirmek
zorunda; "aday yok" da geçerli bir sonuç ama artık kayda geçiyor, sessizce
atlanmıyor.

`postmortem` ana zincirin parçası değildir — tek görevde çalıştırmak anlamsızdır,
birkaç görev biriktikten sonra çağrılır ve çıktısı bir sonraki `atdd`'yi besler.
**2026-09-12'den itibaren `commit`'in kendi 11b adımı bu kararı zorunlu hale
getirdi** (32 tamamlanmış görevde `postmortem` 0/32 çalışmıştı — tetikleme
tamamen hatırlamaya bırakılmıştı): her `commit` sonrası biriken görev sayısı
kontrol edilir, ≥5 ise kullanıcıya çalıştırma sorulur, <5 ise sessizce
atlanmaz, "eşik dolmadı (N/5)" diye kayda geçer.
`threat-model` is conditional: run it when the task touches auth, user input,
file upload/download, payments, personal data, a multi-tenant boundary, or a
new HTTP endpoint. Otherwise note "tetikleyici yok" and go straight to `plan`.
It sits **before** `plan` on purpose — a security requirement that doesn't
exist by `test-copilot`'s turn never gets a test, because `test-copilot` only
tests what atdd.md's acceptance criteria say.

**Bu koşul artık `atdd`'nin kendi 5b adımında ZORUNLU olarak sorulur**
(2026-09-12 — 32 tamamlanmış görev geriye taranınca `threat-model`'in
0/32 çalıştığı, aralarında tam tetikleyici örneği olan auth/webhook
görevlerinin de bulunduğu görüldü; karar tamamen orkestratörün hatırlamasına
bırakılmıştı). `atdd.md`'nin frontmatter'ında `threat_model: done` veya
`threat_model: not-applicable` (gerekçeli) olmadan atdd taslağı tamamlanmış
sayılmaz — bkz. `atdd` SKILL.md 5b. Bu, "koşullu ama hatırlanırsa" değil
"koşullu ama karar kaydı zorunlu" hale getirir.
`dynamic-pentest` da benzer şekilde koşulludur: `red-team` bittikten sonra,
atdd.md'de `threat-model`'in ürettiği gerçek `AC-S<n>` kriteri varsa
(auth/ödeme/multi-tenant/dosya yükleme gibi kanıtlanmış risk) devreye
girer — `red-team`'in "muhakeme ile bulma"sını, gerçek bir exploit
denemesiyle "kanıtlama"ya tamamlar. Harici bir CLI/servis (eski `strix-scan`,
Strix/Docker/OmniRoute) YOK — bunu doğrudan bir Claude subagent'ı (Sonnet/
Opus, gerçek Bash/curl erişimiyle) yapıyor, ekstra kurulum/kimlik doğrulama
bağımlılığı yok. atdd.md'de hiç `AC-S` yoksa (`threat_model: not-applicable`)
bu adım tamamen atlanır, kullanıcı özellikle isterse yine elle çağrılabilir.
Kullanıcı onayı olmadan hedefe yöneltilmez (bkz. `dynamic-pentest`'in kendi
"Yapmayacakların" bölümü).

`frontend-pipeline` da benzer şekilde koşulludur — görev bir UI/component/sayfa
değiştiriyorsa `plan`'dan hemen önce/yanında devreye girer (Discover/Audit/
Design Direction), `verify`'a çok-viewport `vision-test` doğrulaması ekler.
Tetikleyici yoksa (backend-only görev) atlanır, ana zincir değişmez.
**2026-09-12'den itibaren `plan`'ın kendi 0. adımı bu kararı zorunlu hale
getirdi** (32 tamamlanmış görevde `frontend-pipeline` 0/32 çalışmıştı, hatta
"Görsel/UI kriteri" dolu görevlerde bile dedike `vision-test` yerine ad-hoc
Playwright okuması kullanılmıştı) — `plan` artık kendi keşfine başlamadan
önce bu tetikleyiciyi sormak zorunda, sessizce atlayamıyor.
Test-first kuralı geçerlidir (bkz: `pipeline-rules.md` Kural 3). Artık test-first çalışır.

| Step | Reads | Writes | Never does |
|---|---|---|---|
| `jira-sync` | Jira issue, Saga | `saga_task_id` + issue summary (returned to atdd, not a file) | Write atdd.md itself |
| `atdd` | jira-sync output (if any) | `atdd.md` | Talk to Jira/Saga directly, chain to plan |
| `threat-model` | `atdd.md` | `AC-S<n>` criteria appended into `atdd.md` | Write code or tests; pad the list with generic advice |
| `plan` | `atdd.md`, real codebase (read-only) | `plan.md` | Write implementation/test code |
| `test-copilot` | `atdd.md`, `plan.md` | test files (via Haiku sub-agent) + `test_diff.md` | Run tests, write implementation |
| `code-copilot` | `atdd.md`, `plan.md`, `test_diff.md` | implementation files (via Haiku sub-agent) + `code_diff.md` | Write code itself, write/edit tests |
| `verify` | `code_diff.md`, `test_diff.md` | `verify_report.md` | Write/fix code or tests |
| `security-scan` | `code_diff.md` (changed files = scan scope) | `security_scan.md` | Fix findings; call a `MISSING`/`ERROR` gate a pass |
| `refactor` | `code_diff.md`, yeşil test sonucu | `refactor.md` + iyileştirilmiş kod | Testleri değiştirmek, yeni davranış eklemek, kapsam dışı dosyaya dokunmak |
| `red-team` | `atdd.md`, `code_diff.md`, `verify_report.md` | `red_team.json` | Write/fix code |
| `dynamic-pentest` | `atdd.md`'nin `AC-S<n>` kriterleri (threat-model'den), gerçek çalışan hedef veya doğrudan çağrılabilir fonksiyon | `dynamic_pentest.json` | Kod yazmak/düzeltmek; kullanıcı sahibi olmadığı bir hedefe onaysız yönelmek |
| `agentic-judge` | `atdd.md` (agentic AC), gerçek konuşma transkriptleri | `agentic_judge/<conversation_id>.json` | Kod/prompt değiştirmek; toplu istatistik üretmek (bkz. postmortem) |
| `commit` | `red_team.json` (+ varsa `agentic_judge/*.json`) verdict | git commit + push (with user approval) | Run without explicit "commit"/"push" request |
| `postmortem` | tüm görevlerin `red_team.json`'ları | `POSTMORTEM.md` (üzerine eklenir) | Geçmiş görevi değiştirmek, az veriden kalıp uydurmak |

## Her zaman aktif
- `caveman` (full) — tüm yanıtlarda zorunlu, banner ile teyit et.
- `ponytail` (full) — tüm kod kararlarında zorunlu, banner ile teyit et.
Kaynak: `.agents/skills/` (hem CLI ajanı hem Copilot'ta kurulu, bkz. `skills-lock.json`).

## Faz bazlı destek skilleri (pointer — zinciri değiştirmez)

| Faz | Skill | Kural |
|---|---|---|
| Planlama (`plan`'dan önce/yanında) | `frontend-pipeline` | Görev bir UI/component/sayfa değiştiriyorsa (atdd.md'de Görsel/UI kriteri doluysa) `plan`'a girmeden önce çağır — Discover (Playwright) → Audit (`better-*`) → Design Direction (`frontend-design`) sırasını `plan.md`'ye girdi olarak ekler, `verify`'a çok-viewport `vision-test` doğrulaması bağlar. |
| Planlama (`atdd`, `plan`) | `superpowers:brainstorming` | Kapsam belirsizse atdd sorularından önce çağır. |
| Planlama | `superpowers:writing-plans` | `plan.md` bu şablona göre yazılır. |
| Planlama | `wayfinder` (opsiyonel) | İş tek session'a sığmıyorsa plan.md yerine/yanında kullan. |
| Geliştirme (`test-copilot`, `code-copilot`) | `superpowers:subagent-driven-development` | Çok adımlı işi az agent + çok task deseniyle dağıt. |
| Geliştirme | `handoff` (opsiyonel) | Session iş bitmeden kapanırsa HANDOFF.md yaz. |
| Review (`verify`, `red-team`) | `superpowers:requesting-code-review` | `red-team` ile çakışırsa `red-team` esastır (proje-özel adım). |
| Review (`verify` gate 12) | `vision-test` | Görev bir web UI render ediyorsa "Görsel regresyon" gate'i bunu çağırır — Claude yerine Codex'in vision modeli (`codex exec -i`, read-only) ekran görüntüsünü okur, Claude sadece JSON'u okur. |
| Review (`verify` gate 7/8/9/13) | `frontend-audit` | Web UI görevlerinde Playwright (e2e/UX), Lighthouse (performans/erişilebilirlik, gate 8/9) ve ZAP (DAST, gate 13) araç referansı ve doğrulanmış tuzakları burada — `verify` bu skill'i çağırmaz, sadece araçlarını kullanır, karar `verify`'nin kendi gate mantığında kalır. |
| Review | `superpowers:systematic-debugging` | verify gate kırmızıysa kör deneme yerine bunu izle. |
| Geliştirme (`test-copilot`) | `authz-test` | `threat-model` bir yetkilendirme `AC-S` ürettiyse çağır — saldırgan test matrisini `test-copilot`'a extra_instructions olarak verir, canlı sondayı kendi atar. |
| Review (`verify` güvenlik gate'i) | `security-scan` | Deterministik tarayıcılar (sızıntı/SAST/bağımlılık). `red-team`'in yerini almaz: araç bilinen imzayı, red-team iş mantığı açığını görür. |
| Review (`verify`/`security-scan` yanına) | Trail of Bits plugin'leri (`insecure-defaults`, `static-analysis`, `differential-review`, `sharp-edges`, `semgrep-rule-creator`, `variant-analysis`, `supply-chain-risk-auditor`, `mutation-testing`, `fp-check`, `audit-context-building`) | Gerçek bir güvenlik firmasının kendi audit araçları — `security-scan`'in üstüne CodeQL/Semgrep/fail-open tespiti/mutation testing ekler. `fp-check` yanlış-pozitifleri elemek için, `red-team`'e geçmeden kullan. |
| Review (`verify`/`frontend-audit` yanına) | `web-quality-skills` (Addy Osmani) | Lighthouse + Chrome DevTools MCP ile ölçüme dayalı performans/Core Web Vitals/accessibility/SEO denetimi — `frontend-audit`'ten daha olgun, `frontend-pipeline`'ın Validate adımında kullan. |
| Review (`red-team`'in yanına koşullu) | `dynamic-pentest` | **2026-09-12'den itibaren koşullu zincir üyesi**, **2026-09-21'de Strix/Docker/OmniRoute bağımlılığından arındırıldı** (bkz. yukarıdaki The chain) — `threat-model` gerçek `AC-S<n>` kriteri ürettiyse (auth/ödeme/multi-tenant/dosya yükleme gibi gerçek risk), `red-team` bittikten sonra otomatik olarak önerilir/çalıştırılır: dinamik pentest'i doğrudan bir Claude subagent'ı (gerçek Bash/curl erişimiyle) yapar, bulgunun GERÇEKTEN sömürülebilir olduğu kanıtlanır. `AC-S` yoksa hâlâ tamamen opsiyonel/talep üzerine kalır — maliyet/süre (birkaç dakika) yüzünden risksiz görevlerde zorunlu değil. |
| Review (`verify` gate — bağımlılık/IaC) | `trivy-scan` | Bağımlılık CVE'si, secret, yanlış yapılandırma (IaC), lisans taraması — yerel dizin/container image/uzak repo. `security-scan`'in Python-özel SAST'ının (bandit) yerini almaz, tamamlar. |
| Review (`verify` gate 3 — DB dokunulduysa ZORUNLU) | `supabase-check` | Supabase tablosuna dokunan HER değişiklikte çağrılır, atlanmaz — migration'ın gerçekten uygulandığını ve gerçek isteklerin (header/payload) çalıştığını doğrular. Kod yazmaz. |
| Geliştirme (artifact/rapor üretilirken) | `theme-factory` | Bir artifact/rapor/slide'a tutarlı bir renk+font teması uygulanacaksa (10 hazır tema veya özel tema) kullan — kod tabanının kendi tasarım sistemini değiştirmez, sadece üretilen dokümanlar için. |
| Review (`verify`/`vision-test`'e alternatif) | `webapp-testing` (Anthropic) | Local web app'i Python Playwright ile test eder — BU MAKİNEDE önce Browser pane (`mcp__Claude_Browser__*`) tercih edilir (kurulum gerektirmez), bu skill sadece Browser pane'in karşılayamadığı bir ihtiyaçta devreye girer (bkz. skill'in kendi üstündeki not). |
| Review | `superpowers:finishing-a-development-branch` | `commit`'ten hemen önce branch temizliği için. |

## How to use this skill

- **User asks "run the whole thing" / "baştan sona çalıştır":** walk the
  chain top to bottom, calling each skill in turn via the `Skill` tool,
  stopping between steps only for what each step's own rules require
  (clarifying questions in `atdd`, open questions in `plan`, approval in
  `commit`). Don't skip a step to save time — each one exists because
  skipping it caused a real problem before (see each skill's own "Why this
  exists").
- **User asks "nerede kaldık" / "what's next":** call `jira-sync`'s Saga
  lookup (step A) to find the task's current status, then check which
  artifact files exist under `obss_project/artifacts/<task-slug>/`
  (`atdd.md`, `plan.md`, `code_diff.md`, test files, `verify_report.md`,
  `red_team.json`) to see how far the chain already got. Resume at the next
  missing artifact, not from scratch.
- **A step's gate fails** (red verify gate, red-team `block` verdict): don't
  advance to the next step. Report the failure and let the user decide
  whether to retry the failing step or address it manually.
- **Sohbet çok şişti ama görev hâlâ commit'lenmedi (context maliyeti):**
  Görev ortasında (henüz `commit` gelmeden) sohbetin aşırı uzadığını fark
  edersen — çok sayıda tur, uzun debug/keşif turları, tekrarlayan büyük tool
  çıktıları — kullanıcıya şunu öner: mevcut ilerlemeyi (hangi artifact'lar
  tamam, sıradaki adım ne) Saga task'ına (`mcp__saga__task_update` ile
  description/status) yaz, sonra sohbeti temizleyip (`/clear`) yeni bir
  sohbette Saga kaydından + `obss_project/artifacts/<task-slug>/` altındaki
  dosyalardan devam etmeyi teklif et. Otomatik yapma — sadece öner ve
  kullanıcı onaylarsa Saga'yı güncelle. Bu, `commit`'in 12. adımındaki
  (görev bittiğinde clear önerisi) erken/ara sürümüdür — görev bitmeden de
  aynı mantık geçerli olabilir.

## Rule
- This skill never writes code, tests, or reports itself — it only calls
  other skills via the `Skill` tool and reports where things stand.
- `commit` is never auto-invoked as part of "run the whole pipeline" — even
  in full-pipeline mode, stop after `red-team` and ask before committing.
- Destek skilleri (yukarıdaki tablo) otomatik tetiklenmez — `caveman`/`ponytail`
  dışındakiler ihtiyaç anında elle çağrılır, ana zinciri (atdd→...→commit)
  değiştirmez.
