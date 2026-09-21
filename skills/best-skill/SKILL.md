---
name: best-skill
description: Verilen bir script/araç/uygulama/iş akışının gerçek kullanım şeklini, kullanıcıya soru sormadan, bizzat çalıştırıp test ederek çıkarır ve `best-skill-report.md` üretir. Kullanıcıya hiçbir netleştirme sorusu SORULMAZ — belirsizlik varsa kod okunarak/denenerek çözülür. Gerçek çalıştırma/test adımı ana ajanda değil bir subagent'ta yapılır (bkz. proje hafızası: kod/test adımları subagent'a yaptırılır). "şu script'i incele", "bu aracın en iyi kullanım şeklini çıkar", "bu iş akışını test ederek dokümante et" isteklerinde kullan.
---

# Best Skill

> **KURAL: Hiçbir netleştirme sorusu kullanıcıya sorulmaz.** Hedef script/araç
> belirsizse (örn. birden fazla aday dosya), en olası adayı kendin seç ve
> seçimini raporun başında tek satırla gerekçelendir — `AskUserQuestion` bu
> skill'de KULLANILMAZ.

> **KURAL: Gerçek çalıştırma/test işi ana ajanda yapılmaz.** Bu skill,
> incelenecek hedefi ve test planını hazırlar, ardından gerçek komut
> çalıştırma + gözlem + rapor doldurma işini `general-purpose` (veya kod tabanı
> büyükse `Explore` + `general-purpose` kombinasyonu) bir subagent'a
> dispatch eder. Ana ajan sadece subagent'ın sonucunu okuyup kullanıcıya sunar.
> Sebep: bu tür bir iş çok sayıda deneme-yanılma komutu üretir (proje hafızası
> `feedback_subagent_verification_side_effects.md` ile aynı sınıf risk) — ana
> bağlamı komut çıktılarıyla kirletmemek ve yan etkileri izole etmek için
> ayrı bir subagent bağlamında yapılır.

## Adımlar

1. **Hedefi belirle.** Kullanıcının mesajından hedef script/araç/CLI/uygulamayı
   çıkar. Birden fazla aday varsa (örn. aynı isimli birkaç dosya) en olası
   olanı kendin seç — sormadan.
2. **Kapsamı ve ortamı hızlıca gözden geçir** (ana ajanda, hafif): dosya var mı,
   hangi dilde/runtime'da, bağımlılık dosyası var mı (requirements.txt,
   package.json vb.), README/docstring/`--help` çıktısı var mı. Bu adım sadece
   subagent'a doğru bağlamı vermek içindir — derin test burada yapılmaz.
2b. **Kurulum gerekiyorsa BİR KEZ yap, kalıcı işaretle — her çalıştırmada
   tekrar kurulum YAPILMAZ.**
   - Hedefin yanında (aynı klasörde) `.best-skill-setup-done` adlı bir işaret
     dosyası var mı kontrol et.
   - **Varsa:** kurulum adımı tamamen atlanır, doğrudan 3. adıma geç.
   - **Yoksa ve bağımlılık eksikse** (örn. Python projesinde venv yok,
     `requirements.txt`'teki paketler kurulu değil; Node projesinde
     `node_modules` yok): proje türüne uygun **izole** bir kurulum yap —
     Python için `venv` (proje hafızası `feedback_python_venv_izolasyonu.md`:
     asla global `pip install` yapma, diğer projeleri bozar), Node için
     `npm install` (proje kökünde, global değil). Kurulum komutunu ve
     çıktısını rapora "## 0. Setup" bölümü olarak ekle.
   - Kurulum başarıyla bittiğinde `.best-skill-setup-done` dosyasına kurulum
     tarihini ve komutunu tek satır yaz. Bu dosya, aynı hedef için sonraki
     `/best-skill` çağrılarının kurulum adımını atlamasını sağlar — kurulum
     maliyeti sadece ilk seferde ödenir.
   - Bağımlılık zaten kuruluysa (venv/node_modules mevcut ama işaret dosyası
     yoksa) kurulum yapma, sadece işaret dosyasını oluştur.
3. **Test planı çıkar** (ana ajanda): en az şu senaryo sınıflarını kapsayan bir
   liste hazırla:
   - Doğru/beklenen girdiyle çalıştırma (happy path)
   - Eksik/hatalı argüman veya girdi
   - Uç değerler (boş dosya, çok büyük girdi, özel karakterler — ilgiliyse)
   - Tekrarlı/ardışık çalıştırma (idempotency, yan etki birikimi var mı)
   - Hata durumunda çıktı/exit code/log davranışı
   - (Varsa) farklı flag/parametre kombinasyonları
4. **Subagent'a dispatch et — gerçek çalıştırma burada olur.**

   ```
   Agent({
     description: "Test <hedef> gerçek kullanım şekli",
     subagent_type: "general-purpose",
     run_in_background: false,
     prompt: "
       Hedef: <dosya yolu / komut>.
       Bağlam: <2. adımdaki bulgular — dil, bağımlılıklar, --help çıktısı>.
       Görev: aşağıdaki senaryoları GERÇEKTEN ÇALIŞTIRARAK test et, kullanıcıya
       hiçbir soru sorma, tahmin yürütme — sadece gözlemlediğini raporla.
       Senaryolar: <3. adımdaki liste>.
       Her senaryo için: çalıştırdığın tam komut, tam çıktı (stdout+stderr),
       exit code, ve 'başarılı/başarısız/beklenmedik' sınıflandırması kaydet.
       GIT YASAK: commit atma, dosya silme, push yapma. Sadece oku/çalıştır/gözlemle.
       Sonucu şu alanlarla JSON/markdown olarak döndür: name, purpose, what_it_does,
       requirements (runtime/deps/env/files/other), recommended_usage (komut),
       inputs (tablo: input/required/description), expected_output,
       works[] (çalışan durumlar), fails[] (durum + neden), errors_table
       (error/cause/solution), best_practice (komut + yapılması/kaçınılması
       gerekenler), verification (komut + beklenen sonuç), evidence
       (tests/successful/failed sayıları).
     "
   })
   ```

5. **Subagent sonucunu raporla.** Dönen bulguları aşağıdaki şablona birebir
   yerleştirerek `best-skill-report.md` olarak kaydet (proje kökü altında,
   veya kullanıcı başka bir yol belirttiyse orada). Subagent'ın gözlemlemediği
   bir alanı icat etme — "test edilmedi" yaz.
6. Kullanıcıya dosya yolunu ver. Bu skill Hard Stop içermez (salt-okunur bir
   inceleme raporu ürettiği için ATDD'deki gibi onay beklemeye gerek yok),
   ama kullanıcı ek senaryo isterse 3-4. adımları tekrar çalıştır.

## Rapor Şablonu (`best-skill-report.md`)

```markdown
---
target: <script/araç adı ve yolu>
tested_by: subagent (general-purpose)
date: <YYYY-MM-DD>
---

# Best Skill Report — <hedef>

## 0. Setup (varsa)
<İlk çalıştırmada yapılan kurulum komutu + sonucu. `.best-skill-setup-done`
zaten varsa: "Kurulum daha önce yapılmış (<tarih>), atlandı.">

## 1. Overview
**Name:** <ad>
**Purpose:** <ne için var>
**What it does:** <1-2 cümle>

## 2. Requirements
| Alan | Değer |
|---|---|
| Runtime | |
| Dependencies | |
| Environment | |
| Files | |
| Other | |

## 3. Usage

### Recommended
```bash
# en doğru kullanım
```

### Inputs
| Input | Required | Description |
|---|---|---|
| | | |

### Output
```text
# beklenen çıktı
```

## 4. Tested Behavior

### Works
- <senaryo> → <gözlemlenen sonuç>

### Fails
- <senaryo> → <neden>

## 5. Errors & Recovery
| Error | Cause | Solution |
|---|---|---|
| | | |

## 6. Best Practice
En güvenilir kullanım:
```bash
# best practice
```
Kullanırken:
- ...
Kaçınılması gerekenler:
- ...

## 7. Verification
```bash
# verification komutu
```
Beklenen:
```text
# expected result
```

## 8. Evidence
**Tests:** <n>
**Successful:** <n>
**Failed:** <n>

> Bu rapordaki kurallar, `general-purpose` subagent'ın gerçek çalıştırma
> sonuçlarına dayanır (bkz. `atdd`/`red-team`'in "gerçek kanıt" ilkesi).
> Ana ajan bu sonuçları yorumlamadan/icat etmeden aktarır.
```

## Kural
- Kullanıcıya **hiçbir** netleştirme sorusu sorulmaz — belirsizlik kod
  okunarak veya subagent'ın denemesiyle çözülür.
- Gerçek çalıştırma/test **her zaman** subagent'ta yapılır, ana ajan
  bağlamında canlı komut yığını biriktirilmez.
- Subagent'a her dispatch'te **git yasağı** açıkça yazılır (bkz. proje
  hafızası `feedback_haiku_subagent_git_yasagi.md` — izinsiz commit/silme
  riski gerçek bir olaydan geliyor).
- Rapor sadece gözlemlenen davranışı içerir; "muhtemelen şöyle çalışır"
  gibi tahmine dayalı satır yazılmaz — test edilmediyse "test edilmedi" yazılır.
