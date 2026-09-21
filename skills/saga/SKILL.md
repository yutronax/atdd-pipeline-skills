---
name: saga
description: Saga MCP (`mcp__saga__*`) için tam araç referansı + tek-komut giriş noktası. "/saga 255 çalıştır", "saga #254'ten devam" gibi bir Saga task ID'si verildiğinde onu doğrudan bulur (task_get), proje kökünü çözer ve atdd→plan→test-copilot→code-copilot→verify→red-team→commit zincirini o task için başlatır. code/test yazımında Codex ikincil plana alınır, Claude subagent'ı (bağımsız red-team doğrulamasıyla) öncelikli çalışır — bkz. "Codex İkincil Plan" bölümü. Jira'ya bağlı işler için jira-sync kullan, bu skill saf Saga (Jira'sız) akışlar içindir.
---

# Saga — MCP araç referansı + tek-komut giriş noktası

## Kullanılacak destek skiller
- `caveman`, `ponytail` (full) — her zaman aktif.
- `pipeline` — bu skill sadece giriş noktası; task çözüldükten sonra asıl
  zincir (`atdd`→...→`commit`) `pipeline`'ın tarif ettiği sırayla ilerler.
- `superpowers:brainstorming` — atdd adımına geçmeden önce kapsam hâlâ
  belirsizse.

## Why this exists
`jira-sync` sadece Jira'ya bağlı görevler için var (KAN-xx, OBSS Bridge
projesi). Ama Saga'da Jira'sız, doğrudan Saga'da açılmış onlarca task var
(windows-ai-files, linkedin, avatar-demo, kullanici-profili projeleri) ve
bunlar için "saga #254'ten devam" dendiğinde her seferinde hangi
`mcp__saga__*` aracının hangi parametreyi aldığını yeniden keşfetmek
gereksiz tekrardır. Bu skill iki şeyi sabitliyor: (1) her Saga aracının
gerçek şeması + çalıştırma kodu, (2) bir task ID'sinden pipeline'ı
başlatana kadarki çözümleme adımları.

KURAL: Codex CLI kotası tükenmiştir. Bu skill üzerinden başlatılan
görevlerde `test-copilot`/`code-copilot`'un normal Codex-önce sırası
bölümüne bak.

## Ne zaman bu skill çağrılır
- Kullanıcı `/saga <id>` veya `/saga <id> çalıştır` yazdığında.
- "saga #<id>'ten devam", "saga task <id>'i çalıştır", "<id> numaralı Saga
  görevini yap" gibi bir Saga task ID'si doğrudan verildiğinde.
- Jira ID'si YOKSA ve iş doğrudan Saga'da yaşıyorsa. Jira ID'si varsa
  (KAN-xx) bunun yerine `jira-sync`'i çağır.
- Saga'da genel gezinme/arama/rapor ihtiyacı olduğunda (örn. "dashboard'u
  göster", "geçen haftadan beri ne değişti") — aşağıdaki araç referansından
  doğrudan ilgili aracı çağır, pipeline'ı tetikleme.

---

## Bölüm 0 — Saga MCP bağlanamıyorsa (önce buraya bak)

`mcp__saga__*` deferred listede yoksa ve `ToolSearch({query: "mcp__saga__"})`
"No matching deferred tools found" dönerse, sorun neredeyse hep şudur:

**Saga MCP global bir `.mcp.json` ile değil, `~/.claude.json` içindeki
proje-özel `projects.<proje-yolu>.mcpServers` girişiyle tanımlanır.**
Sabit blok:
```json
"saga": {
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "saga-mcp"],
  "env": {
    "DB_PATH": "C:\\Users\\YUSUF ÇİNAR\\.gemini\\antigravity\\memory\\saga.tracker.db"
  }
}
```
Tüm projeler aynı `DB_PATH`'i (tek ortak `saga.tracker.db`) paylaşır — proje
ayrımı dosya düzeyinde değil, Saga içindeki `project_id` alanıyla yapılır.

Kontrol/onarım adımları:
1. `~/.claude.json` dosyasını aç, `projects` altında geçerli proje dizinine
   bak. **Aynı klasörün `\` (backslash) ve `/` (forward slash) yazımı iki
   AYRI proje kaydı yaratır** — biri `saga` girişini içerebilir, diğeri
   içermeyebilir (ya da `mcpServers` hiç yoktur, ya da `{}` boştur). Hangi
   normalize yol açıksa (Windows'ta genelde backslash) onu bul.
2. `saga` bloğu eksikse veya boşsa, yukarıdaki sabit bloğu o proje kaydına
   (varsa **her iki yol-varyantına da**, hangisinin aktif kullanılacağı
   belirsizse) ekle.
3. **KURAL: Bu dosyayı düzenlemek mevcut oturumda anında etkili olmaz.**
   Claude Code MCP sunucu listesini oturum başlangıcında okur. Config
   değişikliğinden sonra kullanıcıya oturumu kapatıp aynı klasörde yeni bir
   Claude Code oturumu başlatmasını söyle — sessizce "düzelttim, çalışıyor
   olmalı" deyip aynı oturumda tekrar iş yapmaya çalışma, bir sonraki
   `ToolSearch` denemesi yine boş döner ve kullanıcı aynı hatayı tekrar
   yaşar.
4. Yeni oturumda `ToolSearch({query: "mcp__saga__", max_results: 40})` ile
   doğrula, sonra Bölüm B'ye geç.

Bu kural olmadan her yeni sohbette "saga bağlanmıyor" hatası sıfırdan
keşfedilip aynı düzeltme tekrar tekrar yapılır — bu bölüm o keşfi atlamak
için var.

---

## Bölüm A — Saga MCP Araç Referansı (tam çalıştırma parametreleriyle)

Aşağıdaki tüm araçlar `mcp__saga__*` altında canlı ve yüklü. Deferred
listede görünüyorlarsa önce `ToolSearch({query: "mcp__saga__", max_results: 40})`
ile tek seferde hepsini yükle — tek tek `select:` yapma.

### Proje

```
mcp__saga__project_list({ status?: "active"|"on_hold"|"completed"|"archived" })
mcp__saga__project_create({ name, description?, status?: "active" (default), tags?: [] })
mcp__saga__project_update({ id, name?, description?, status?, tags? })
```

### Epic

```
mcp__saga__epic_list({ project_id, status?, priority?, branch? })
mcp__saga__epic_create({ project_id, name, description?, status?: "planned" (default), priority?: "medium" (default), tags?: [], branch? })
mcp__saga__epic_update({ id, name?, description?, status?, priority?, tags?, sort_order?, branch? })
```
`branch: "current"` aktif git branch'ini otomatik algılar; boş string ile
branch-agnostic (global) epic'lere daraltılır.

### Task

```
mcp__saga__task_list({ epic_id?, status?, priority?, assigned_to?, tag?, branch?, sort_by?: "priority" (default)|"created"|"due_date"|"status", limit?: 50 (default) })
mcp__saga__task_get({ id })
mcp__saga__task_create({ epic_id, title, description?, status?: "todo" (default), priority?: "medium" (default), assigned_to?, due_date?, estimated_hours?, depends_on?: [id...], source_ref?: { file, repo?, commit?, line_start?, line_end? }, tags?: [] })
mcp__saga__task_update({ id, title?, description?, status?, priority?, assigned_to?, due_date?, estimated_hours?, actual_hours?, depends_on?, source_ref?, tags?, sort_order? })
mcp__saga__task_batch_update({ ids: [id...], status?, priority?, assigned_to? })
```
`task_get` en zengin çağrı — subtasks, notes, comments, dependencies dahil
tek seferde döner. Yeni bir task'a bakarken önce bunu çağır.

`status` geçişleri (`task_update`) otomatik `activity_log`'a yazılır — ayrı
bir log kaydı atmana gerek yok.

### Subtask

```
mcp__saga__subtask_create({ task_id, titles: "tek başlık" | ["başlık1", "başlık2", ...] })
mcp__saga__subtask_update({ id, title?, status?: "todo"|"in_progress"|"done", sort_order? })
mcp__saga__subtask_delete({ ids: id | [id, ...] })
```

### Yorum (Comment)

```
mcp__saga__comment_add({ task_id, content, author? })
mcp__saga__comment_list({ task_id })
```

### Not (Note)

```
mcp__saga__note_save({ title, content, id?, note_type?: "general"(default)|"decision"|"context"|"meeting"|"technical"|"blocker"|"progress"|"release", related_entity_type?: "project"|"epic"|"task", related_entity_id?, tags?: [] })
mcp__saga__note_list({ related_entity_type?, related_entity_id?, note_type?, tag?, limit?: 30 (default) })
mcp__saga__note_search({ query, note_type?, limit?: 20 (default) })
mcp__saga__note_delete({ id })
```
`note_save`'e `id` verirsen günceller, vermezsen yeni not oluşturur — ayrı
bir update aracı yok.

### Şablon (Template)

```
mcp__saga__template_list({})
mcp__saga__template_create({ name, description?, tasks: [{ title, description?, priority?, estimated_hours?, tags? }, ...] })
mcp__saga__template_apply({ template_id, epic_id, variables?: { key: value, ... } })
mcp__saga__template_delete({ id })
```
`tasks[].title`/`description` içinde `{variable}` placeholder kullanılabilir,
`template_apply`'daki `variables` ile doldurulur.

### Tracker (genel bakış / arama / import-export)

```
mcp__saga__tracker_init({ project_name?, project_description? })   // DB boşsa yeni proje kurar, doluysa mevcut projeyi döner
mcp__saga__tracker_dashboard({ project_id?, branch? })              // tek çağrıda: proje + epic'ler + istatistik + son aktivite + son notlar
mcp__saga__tracker_search({ query, entity_types?: ["project"|"epic"|"task"|"note", ...], limit?: 20 (default), branch? })
mcp__saga__tracker_session_diff({ since })                          // ISO datetime — son session'dan beri ne değişti
mcp__saga__tracker_export({ project_id? })                          // tam proje JSON dökümü (backup/migration)
mcp__saga__tracker_import({ data })                                 // tracker_export formatındaki JSON'u yeni ID'lerle içe aktarır
```

### Aktivite Günlüğü

```
mcp__saga__activity_log({ entity_type?: "project"|"epic"|"task"|"subtask"|"note", entity_id?, action?: "created"|"updated"|"deleted"|"status_changed", since?, limit?: 50 (default) })
```

---

## Bölüm B — `/saga <id>` giriş noktası: task'tan pipeline'a

### 1. Task'ı çöz
```
mcp__saga__task_get({ id: <id> })
```
Döner: `title`, `description`, `status`, `priority`, `epic_id`, `epic_name`,
`subtasks`, `notes`, `comments`, `depends_on`, `dependents`.

- `status: "done"` ise: kullanıcıya söyle, "yeniden mi çalıştırayım/sadece
  doğrulayayım mı" diye sor — sessizce baştan başlama.
- `status: "blocked"` ise: `comment_list({task_id})` ve `note_list({
  related_entity_type: "task", related_entity_id: id})` ile blokaj
  nedenini oku, kullanıcıya özetle, onunla birlikte karar ver.
- `depends_on` doluysa: bağımlı task'ların durumunu (`task_get` her biri
  için) kontrol et — hâlâ `todo`/`in_progress` olan bir bağımlılık varsa
  kullanıcıyı uyar, yine de devam edip etmeyeceğini sor.

### 2. Proje kökünü çöz

**Önce bilinen tabloya bak** (aşağıda) — task_get `project_id` döndürmez,
sadece `epic_id`/`epic_name`. Bilinen tabloda proje eşleşmiyorsa:
```
mcp__saga__project_list({ status: "active" })
```
sonra her aday proje için
```
mcp__saga__epic_list({ project_id: <aday> })
```
çalıştırıp `epic_id`'nin hangi projede olduğunu bul (proje sayısı azdır,
~5-6 aktif — tam tarama ucuzdur). Bulduğun projenin `description` alanını
oku: bu projede "Proje dizini: <mutlak yol>" kalıbı varsa (kurulan
konvansiyon, bkz. tablo) onu kullan; yoksa `Glob`/`find` ile proje adını
ara ve **bulduğunda projenin Saga description'ını `project_update` ile
"Proje dizini: <yol>" ekleyerek güncelle** — bir sonraki sefer tekrar
aranmasın.

**KURAL:** Bilinen Saga Projeleri aşağıdadır. Daima `project_list` ile doğrula:

| project_id | name | Proje dizini | Artifact konumu |
|---|---|---|---|
| 2 | windows-ai-files | `C:\Users\YUSUF ÇİNAR\OneDrive\Belgeler\Masaüstü\projelerim\Yazılım_müh\windows-ai-files` | `artifacts/<task-slug>/` (proje kökünde, `obss_project` YOK) |
| 6 | LinkedIn Staj Bulma Sistemi | `C:\Users\YUSUF ÇİNAR\OneDrive\Belgeler\Masaüstü\projelerim\linkedin` | `docs/linkedin-staj-bulma-sistemi/` |
| 5 | avatar-demo | `C:\Users\YUSUF ÇİNAR\OneDrive\Belgeler\Masaüstü\projelerim\test bu\avatar-demo` | proje description'ında yok, ilk kullanımda tespit edip description'a ekle |
| 7 | kullanici-profili | proje description'ında yok, ilk kullanımda tespit edip description'a ekle | — |
| 4 / 3 | OBSS Bridge / OBSS Bridge - ask_claude (arşivlenmiş) | `obss_project/` monorepo (çok-proje, `Group-9/` gibi alt klasörler) | `obss_project/artifacts/<task-slug>/` |
| 1 | Antigravity Hub (arşivlenmiş) | `.claude`/`.gemini` altındaki scripts/hooks, ayrı repo değil | — |

### 3. Task-slug öner ve onaylat
`atdd`'nin kendi kuralı: kullanıcının isteğinden kısa `kebab-case` slug
türet, onaylat. Burada ek olarak task başlığından (`task_get`'in `title`
alanı) bir öneri türetebilirsin, ama yine de `atdd`'nin kendisi onaylatır —
burada sadece öneriyi hazırla.

### 4. Durumu `in_progress`'e çek
```
mcp__saga__task_update({ id: <id>, status: "in_progress" })
```

### 5. `atdd` skill'ini başlat — `jira-sync`'i atla
Bu task Jira'ya bağlı değilse (`jira_id: null`), `atdd.md` frontmatter'ına
doğrudan `saga_task_id: <id>` yaz, jira-sync'i çağırma. `atdd`'ye
argüman olarak task'ın çözülmüş bağlamını (title, description, epic_name,
priority, proje kökü) ver ki atdd yeniden sorgulamasın.

### 6. Pipeline'ın geri kalanı — `pipeline` skill'inin sırasıyla aynı
```
atdd → (threat-model, koşullu) → plan → test-copilot → code-copilot → verify → (refactor, koşullu) → red-team → commit (kullanıcı onayıyla)
```
**Ama** `test-copilot`/`code-copilot` adımlarında aşağıdaki "Codex İkincil Plan" geçerliyse, o iki skill'i normal haliyle ÇAĞIRMA — bunun yerine Bölüm C'deki akışı uygula.

### 7. Commit sonrası Saga güncellemesi
`commit` skill'i push'u tamamladıktan sonra:
```
mcp__saga__task_update({ id: <id>, status: "done", source_ref: { file: "<ana değişen dosya>", commit: "<commit hash>", repo: "<owner/repo>" } })
```
`jira-sync`'in "Status updates" kuralıyla aynı: sessizce yapma, önce sor
("Saga task #<id>'yi 'done' işaretleyeyim mi?"), kullanıcı onaylarsa
güncelle.

---

## Bölüm C — Codex İkincil Plan

**KURAL:** Codex CLI kotası tükenmiştir. Code/test yazımında Codex ikincil plana alınır.

**Bu skill üzerinden başlatılan her pipeline için:**

- **a. Testleri yaz (red).** `atdd.md` + `plan.md`'yi oku, kabul
  kriterlerinin her satırına karşılık gelen başarısız testleri
  **doğrudan Claude kendisi** (`Write`/`Edit`) yazar — `test-copilot`'un
  "her satır Codex'ten gelir" kuralı bu pencerede bilinçli olarak
  esnetilir. Yazdıktan hemen sonra
  gerçek test komutunu çalıştırıp **kırmızı olduğunu doğrula** —
  doğrulanmamış "muhtemelen fail eder" kabul edilmez.
- **b. Implementasyonu yaz (green).** Testler kırmızıyken, yine Claude
  doğrudan implementasyonu yazar. `code-copilot`'un 2a/2b/2c
  bölümlerindeki CAVEMAN Principles / Definition of Done / Output
  Contract kurallarını kendine uygula (dış Codex'e değil, kendi
  yazımına). Yazdıktan sonra testleri tekrar çalıştırıp **yeşile
  döndüğünü doğrula**.
- **c. Raporları normal şekilde yaz.** `test_diff.md` ve `code_diff.md`
  yine üretilir (formatı `test-copilot`/`code-copilot`'unkiyle aynı),
  üstüne şu notu ekle: *"Codex kotası dolu; bu task'ta test/implementasyon istisnai olarak Claude tarafından yazıldı, kullanıcı onaylıdır."*
- **d. `verify` normal çalışır** — bu adım zaten kimin yazdığından
  bağımsız, gerçek build/test/lint çalıştırıyor.
- **e. `red-team` MUTLAKA bağımsız bir subagent'a yaptırılır**
  (`Agent` tool, `subagent_type: "obss-red-team"`) — bu adım bu
  override modunda pazarlıksız. Claude'un kendi yazdığı kodu yine
  Claude'un (aynı bağlamda) incelemesi güvenilmez sayılır; subagent'a
  gerçek `git diff`'i `code_diff.md`/`test_diff.md` iddialarıyla
  bağımsız karşılaştırmasını ve normal 8 inceleme alanını + scope/risk
  review'u açıkça iste. `verdict: "block"` çıkarsa commit'ten önce
  kullanıcıyı aç aç uyar.
- **f. `AI_DEVLOG.md`'ye (proje kökünde varsa) bu istisnayı kaydet** —
  hangi task, neden Codex atlandı, hangi subagent doğruladı (bkz.
  windows-ai-files `AI_DEVLOG.md`'deki Saga #254 örneği).

**KURAL:** Ana asistan hem yazıp hem aynı bağlamda incelerse doğrulama kör noktası oluşur. Bağımsız bir subagent, gerçek dosyaları sıfır önyargıyla, "yazan bu mu dedi" diye güvenmeden okumalı ve doğrulamalıdır.

---

## Kural
- Bu skill kendisi hiçbir zaman implementasyon/test dosyası yazmaz — o iş
  Bölüm B'nin 6. adımındaki zincire (veya Bölüm C'nin override'ına) ait.
- Saga'ya yazan her adım (`task_update` ile status değişimi, `project_update`
  ile proje dizini ekleme) sessiz yapılabilir (idempotent, düşük riskli)
  **ancak** task'ı `done` işaretlemek her zaman kullanıcı onayı ister
  (`jira-sync`'teki "Status updates" kuralıyla aynı).
- Bilinmeyen bir proje/epic ID'si asla uydurulmaz — `project_list`/
  `epic_list` ile doğrulanmadan bir task'ı yanlış projeye bağlama.
- Codex override kuralı geçici bir kota önlemidir. Kota doluluğu bittiğinde normal akışa dönülmelidir.
