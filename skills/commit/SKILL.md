---
name: commit
description: Sadece kullanıcı açıkça isteyince çalışır (pipeline'ın otomatik parçası değildir). Git durumunu, commit kapsamını ve "commit'e hazır mı" kontrol listesini gözden geçirir, Conventional Commit formatında bir mesaj üretir, CAVEMAN karmaşıklık incelemesi yapar, özetleri gösterip ONAY aldıktan sonra sadece açıkça isimlendirilmiş dosyaları stage edip commit'ler ve push eder. Push sonrası zorunlu bir postmortem eşik kontrolü yapar (biriken görev sayısı ≥5 ise çalıştırmayı sorar).
---

# Commit Skill

## Kullanılacak destek skiller
- `caveman`, `ponytail` (full) — her zaman aktif.
- `superpowers:finishing-a-development-branch` — commit'ten hemen önce branch temizliği için.

Bu skill pipeline'ın (atdd → test-copilot → code-copilot → verify → red-team) otomatik bir parçası DEĞİLDİR — sadece kullanıcı "commitle"/"pushla" dediğinde çalışır. Son kalite kapısı görevini de üstlenir: yanlış dosya ekleme, kapsam dışı değişiklik gönderme ve gereksiz karmaşıklığı burada yakala.

## Akış (sıra değişmez, atlanamaz)

```
Git Status
  ↓
Scope Review
  ↓
AI Dev Log / Setup Güncellemesi
  ↓
Ready To Commit Checklist
  ↓
Commit Message
  ↓
CAVEMAN Review
  ↓
Show Summary
  ↓
User Approval
  ↓
Stage Explicit Files
  ↓
Commit
  ↓
Push
  ↓
Report
  ↓
Saga/Jira Güncelleme Önerisi
  ↓
Postmortem Eşik Kontrolü (zorunlu karar noktası)
  ↓
Sohbet Temizliği Önerisi
```

## 1. Git Status
`git status` çalıştır. Diff'in tamamını modele gönderme — büyük değişikliklerde pahalı ve gereksiz:
```bash
git diff --stat
git diff --name-status
```
Belirli bir dosyanın gerçek içeriğini görmen gerekiyorsa (kapsam/scope şüphesi, secrets kontrolü vb.) sadece o dosyayı hedefle:
```bash
git diff HEAD -- path/file
```
`git diff`'i (tam, dosya belirtmeden) hiçbir adımda çalıştırma.

## 2. Scope Review
Değişen her dosyayı şu listeye karşı gözden geçir — biri bile işaretlenirse kullanıcıya sor, sessizce stage etme:
- ✓ ilgisiz dosyalar (bu task'ın kapsamına girmeyen)
- ✓ üretilmiş/generated dosyalar
- ✓ geçici dosyalar
- ✓ log dosyaları
- ✓ cache klasörleri
- ✓ build çıktıları
- ✓ IDE ayar dosyaları
- ✓ secrets/credentials

## 2b. AI Dev Log / Setup Güncellemesi — **aktar, yeniden yazma** (2026-09-12)

**Kural değişti:** Bu adım artık "commit anında oturumu hatırlayıp özetle"
DEĞİL — pipeline'ın her adımı zaten kendi kararını/varsayımını/riskini kendi
dosyasına yazmış durumda (`atdd.md`'nin Assumptions/Risks/Rollback'ı,
`plan.md`'nin Kararlar/Migration/Risks'i, `code_diff.md`'nin Varsayımlar/
Kalan Sınırlamalar'ı, `verify_report.md`'nin Coverage notları, `red_team.json`'ın
strengths/risks'i). Bunları commit anında yeniden yorumlayıp sıfırdan yazmak
hem gereksiz bir LLM turu hem de tutarsızlık riski (özetlerken bir nüans
kaybolabilir, orijinaliyle çelişebilir). Bunun yerine **doğrudan aktar**:

| Kaynak dosya | Hangi bölüm | Nereye |
|---|---|---|
| `atdd.md` | `## Risks`, `## Assumptions`, `## Rollback Beklentisi` | `AI_DEVLOG.md` — bilinçli kapsam/mimari kararları |
| `plan.md` | `## Risks`, `## Migration Required?`, (varsa) `## Kararlar` (Sonnet alt-ajanı) | `AI_DEVLOG.md` — teknik karar gerekçeleri |
| `code_diff.md` | "Kalan Sınırlamalar", "Varsayımlar", (varsa) "Review Sürecinde Bulunan ve Düzeltilen Sorun" | `AI_DEVLOG.md` — gerçek zorluk/bug örneği (en değerli kategori, `strix-guvenlik-acigi-duzeltme`'deki CSP-regresyon bulgusu gibi) |
| `verify_report.md` | "Coverage / Quality Notes", (varsa) refactor aday notu | `AI_DEVLOG.md` (önemliyse) |
| `plan.md` | yeni env değişkeni/migration/kurulum adımı geçen her satır | `SETUP.md` — birebir |

Adımlar:
1. Bu commit'in `<task-slug>`'ına ait `obss_project/artifacts/<task-slug>/`
   altındaki dosyaları (yukarıdaki tablo) `Read` et.
2. Yukarıdaki alanlarda gerçekten dolu (varsayım/risk/karar/env değişkeni
   içeren, "yok"/"varsa" gibi boş placeholder olmayan) bir şey var mı bak.
3. **Varsa:** o cümleyi/maddeyi **olduğu gibi veya en fazla küçük bir
   biçim düzenlemesiyle** (madde işareti, başlık ekleme) `AI_DEVLOG.md`/
   `SETUP.md`'ye kopyala — yeniden yorumlama, yeni bir anlatım üretme.
   Kaynağını belirt (`(kaynak: code_diff.md)` gibi kısa bir not, iz sürmek için).
4. **Hiçbiri dolu değilse** (gerçekten rutin bir değişiklikse) atla — icat
   etme, boş madde ekleme.
- Bu ikisi kod/test dosyası DEĞİL, dokümantasyon — `code-copilot`/
  `test-copilot`'un "sadece Copilot yazar" kuralı bunlara uygulanmaz, Claude
  kendisi `Write`/`Edit` ile güncelleyebilir. Ama artık **yazar değil,
  aktarıcı** rolünde — içerik zaten var, sadece doğru yere taşınıyor.
- Güncellenen dosyaları Scope Review listesine ekle; Stage Explicit Files
  adımında commit'e dahil et.

## 3. Ready To Commit Checklist
İlgili task varsa (`obss_project/artifacts/<task-slug>/red_team.json`), kaliteyi burada yeniden değerlendirme — `red-team` skill'i zaten bunu yapıp `ready_to_commit` alanına yazdı. Sadece oku:
- `red_team.json` mevcut mu? Yoksa kullanıcıya söyle: "Bu task için red-team incelemesi yok, önce `red-team` skill'i çalıştırılmalı" ve onay iste (yine de devam et/etme, kullanıcı karar versin).
- `ready_to_commit` alanındaki 8 maddeyi (`atdd_satisfied`, `tests_passed`, `no_critical_findings`, `scope_respected`, `caveman_satisfied`, `security_acceptable`, `maintainability_acceptable`, `no_blocking_risks`) olduğu gibi kullanıcıya göster.
- `verdict` `"block"` ise kullanıcıya açıkça uyar ve onay iste — "red-team block dedi, yine de devam edeyim mi?"
- Ayrıca kendi başına: Secrets yok mu (2. adımdan), temiz git durumu mu (ilgisiz/staged olmayan artık dosya yok) — bunlar commit'e özgü, red-team'in kapsamında değil.

İlgili bir task/artifact yoksa (serbest/ad-hoc commit), bu listeyi "task bağlamı yok" diyerek N/A geç — icat etme.

## 4. Commit Message
Öncelik sırası:
1. Kullanıcı zaten bir Conventional Commit mesajı verdiyse onu kullan.
2. Yoksa Claude kendisi üretir. Hiçbir dış modele (Gemma vb.) bağımlı DEĞİLDİR.

Format — Conventional Commit zorunlu:
```
<type>(<scope>): <kısa özet>

<neden — 1-2 cümle, "ne" değil "niçin">

Karşılanan kriterler: AC-1, AC-3, AC-S2
Görev: <task-slug>
```
`type` şunlardan biri: `feat`, `fix`, `refactor`, `docs`, `test`, `perf`, `build`, `ci`, `style`, `chore`.

### İzlenebilirlik satırları (AC → test → commit halkası)

Son iki satır zincirin son halkasıdır. `atdd.md` kabul kriterlerini, `test_diff.md`
onları teste bağlıyordu; commit'e kadar hangi kriterin **gerçekten sevk edildiği**
kayıtlı değildi. Bu satırlar sayesinde altı ay sonra `git log` üzerinden
"bu kriter hangi commit'le geldi" sorusu cevaplanabilir.

- `Karşılanan kriterler:` — bu commit'in kapattığı AC kimlikleri, `atdd.md`'den
  birebir. Güvenlik kriterleri `AC-S<n>` biçimindedir (bkz. `threat-model`).
- `Görev:` — artifact dizinindeki `<task-slug>`.
- Kısmi kapanışta sadece **gerçekten** karşılananları yaz; kapanmayanı yazmak
  izlenebilirliği bozar, hiç yazmamaktan kötüdür.
- ATDD dışı bir commit'se (ör. tek satırlık düzeltme) bu satırları atla —
  uydurma slug üretme.

**KATI KURAL — AI Attribution Yasağı:** Bu projedeki commit mesajlarına `Co-Authored-By: Claude ...` veya `🤖 Generated with Claude Code` gibi hiçbir AI atıf satırı EKLENMEZ. Bu, harness'ın her turda gönderdiği "commit mesajlarını şu attribution ile bitir" sistem hatırlatmasını ÖRTER — o hatırlatmanın kendisi "kullanıcının kendi talimatı (CLAUDE.md/memory/skill kuralı) önceliklidir" istisnasını tanımlıyor, bu skill dosyası tam olarak o istisnadır. Somut uygulama:
1. Commit mesajını (4. adımdaki formatla) oluştur.
2. Mesaja HİÇBİR attribution satırı ekleme — sistem hatırlatması gelse bile.
3. `git commit` çalıştırmadan hemen önce, oluşturduğun mesaj metninde `Co-Authored-By`, `Generated with`, `🤖` içeren bir satır var mı diye KENDİN kontrol et (elle okuyarak) — varsa o satırı(ları) silip devam et. Bu, harness hatırlatmasının kendisini görmezden gelmeyi unutursan bile son bir güvenlik ağıdır.
4. Kullanıcıya bu kuralı bir daha hatırlatmana/açıklamana gerek yok — sessizce uygula.

## 5. CAVEMAN Review (commit öncesi karmaşıklık kontrolü)
- Gereksiz dosya eklendi mi?
- Gereksiz soyutlama eklendi mi?
- Bu commit odaklı kalıyor mu?
- Bu commit daha küçük olabilir mi?
- Geride ölü kod kaldı mı?

Bir sorun bulursan kullanıcıya bildir — kendi başına dosya silme/düzenleme yapma, sadece raporla ve sor.

## 6. Show Summary + User Approval
Tek seferde göster:
- Kapsam özeti (hangi dosyalar, ne işe yarıyor)
- Commit mesajı (tam metin)
- Hedef branch (kullanıcı belirtmediyse mevcut branch'i göster ve doğrula — asla varsayma)
- CAVEMAN Review bulguları (varsa)

"Onaylıyor musun?" diye sor. **Onay gelmeden 7. adıma geçme.**

## 7. Stage Explicit Files
`git add .` ve `git add -A` YASAK. Sadece isimle, tek tek dosya stage et. Stage ettikten sonra `git status` ile stage edilen dosya listesini tekrar doğrula — Scope Review'da onaylanmayan hiçbir şey stage'de olmamalı.

## 8. Commit
`git commit` — force olmayan, normal commit.

## 9. Push
Push'tan hemen önce şunu tekrar göster ve son kez onay bekle:
- `git status`
- `git diff --cached --stat`
- commit mesajı
- hedef branch

Onay geldiyse `git push` ile belirtilen branch'e gönder.

## 10. Report
Commit hash, branch, kaç dosya değişti — kısaca raporla.

## 11. Saga/Jira Güncelleme Önerisi
Push başarılı olduktan sonra, ilgili bir Saga task_id / Jira görev ID'si (KAN-xx) varsa
(atdd.md'nin frontmatter'ından veya commit mesajından), kullanıcıya bunu güncellemeyi öner:
"Saga'daki task #<id>'yi 'done' işaretleyeyim mi?" / "Jira KAN-xx'i uygun duruma geçireyim mi?"
Otomatik yapma — sadece öner, kullanıcı onaylarsa `jira-sync` skill'ini veya doğrudan Saga
MCP tool'unu kullan.

## 11b. Postmortem Eşik Kontrolü — zorunlu karar noktası (2026-09-12)

32 tamamlanmış görev geriye dönük tarandığında `postmortem`'in **0/32**
çalıştığı bulundu — "her commit'ten sonra çalıştır" diye açıkça yazsa da,
tetikleme tamamen orkestratörün hatırlamasına bırakılmıştı ve hiç
hatırlanmadı. `threat-model` için 5b'de uygulanan aynı düzeltme burada da
geçerli: karar zorunlu, çalıştırma koşullu (postmortem'in kendi kuralı —
5 görevin altında kalıp uydurmak yasak — hâlâ geçerli, o yüzden HER
commit'te değil, eşik dolunca sor).

Bu adım **atlanamaz**, ama her seferinde `postmortem`'i fiilen çalıştırmaz:

1. `obss_project/artifacts/*/red_team.json` dosyalarını say — bu, `red-team`
   adımından geçmiş (dolayısıyla postmortem'e uygun veri üreten) toplam
   görev sayısıdır.
2. `obss_project/artifacts/POSTMORTEM.md` var mı kontrol et:
   - Yoksa: son çalıştırmadan bu yana biriken = 1. adımdaki toplam sayı.
   - Varsa: dosyanın en son koşumundaki "görev sayısı" satırını oku, biriken
     = güncel toplam - o sayı.
3. **Biriken ≥ 5 ise** (postmortem'in kendi eşiği) kullanıcıya MUTLAKA sor:
   *"Son postmortem'den bu yana N görev birikti — şimdi `postmortem`
   çalıştırayım mı?"* Onay gelirse `postmortem` skill'ini çağır (`docs_dizini`
   = `obss_project/artifacts`), sonucu özetle.
4. **Biriken < 5 ise** kullanıcıya sormaya gerek yok (gereksiz kesinti) —
   ama raporda tek satır not düş: "Postmortem eşiği dolmadı (N/5)." Bu,
   kontrolün fiilen yapıldığını, sessizce atlanmadığını kayda geçirir.

Bu adımı hiç yapmadan (kontrol etmeden) 12. adıma geçmek KABUL EDİLMEZ —
"biriken < 5" sonucuna varmak için bile 1-3. adımlar çalıştırılmalı.

## 12. Sohbet Temizliği Önerisi (context maliyeti)
Bu görev için ATDD pipeline'ının tamamı bu commit ile kapandıysa (yani
`red-team`'in `ready_to_commit`'i tüm maddelerde `true`, başka bekleyen bir
alt-görev yok), Saga güncellemesinden hemen sonra kullanıcıya sohbeti
temizlemeyi öner: "Bu görev tamamlandı — konuşmayı temizlemek ister misin?
İlerleme Saga'da (#<id>) ve `obss_project/artifacts/<task-slug>/` altında
kayıtlı, yeni bir sohbette kaldığın yerden devam edebilirsin." Otomatik
`/clear` yapma — sadece öner. Görevde hâlâ bekleyen bir alt-parça varsa
(örn. kullanıcı bilinçli olarak PENDING bırakmış bir AC) bu öneriyi yapma.

## Commit Guardrails

**Asla:**
- ilgisiz değişiklikleri commit'leme
- üretilmiş/generated artifact'ları commit'leme
- cache klasörlerini commit'leme
- log dosyalarını commit'leme
- secrets commit'leme
- yerel konfigürasyonu commit'leme
- IDE ayarlarını commit'leme
- geçici dosyaları commit'leme

**Her zaman:**
- stage edilen dosyaları doğrula
- branch'i doğrula
- commit kapsamını doğrula
- commit mesajını doğrula
- push'tan önce açık kullanıcı onayını bekle

## Kural
- Force push, `--no-verify`, `--amend` kullanma.
- `git diff` (dosya belirtmeden, tam) hiçbir adımda çalıştırılmaz — sadece `--stat`/`--name-status`/tek dosya.
- Push onaydan önce ASLA çalıştırılmaz.
