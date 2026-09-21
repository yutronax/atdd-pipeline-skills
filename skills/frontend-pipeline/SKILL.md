---
name: frontend-pipeline
description: UI/UX görevleri için atdd zincirinin genişletilmiş dalı — Discover → Audit → Design Direction → Plan → Implement → Validate → Review → Iterate → Report. Playwright MCP (gerçek tarayıcı keşfi/QA), frontend-design (estetik yön), better-* skilleri (layout/ui/typography/color/accessibility eleştirisi) ve vision-test'i (screenshot doğrulama) tek zincirde sıralar. Görev bir .tsx/.jsx/.vue/.css/component/sayfa değiştiriyorsa veya atdd.md'nin "Görsel/UI kriteri" alanı doluysa `plan` adımından önce/yanında çağır. Kendisi kod yazmaz — hangi adımda hangi skill/araç çalışır, onu sıralar.
---

# Frontend Pipeline — UI/UX görevleri için atdd'nin genişletilmiş dalı

## Neden var
`atdd` zinciri (`plan` → `test-copilot` → `code-copilot` → `verify` → `red-team`)
framework-agnostik — bir UI görevinde "estetik yön ne olacak", "gerçek tarayıcıda
nasıl görünüyor", "layout/typography/renk/erişilebilirlik kuralları ihlal edildi mi"
sorularını sormaz. Bu skill o boşluğu dolduran **koşullu bir dal**dır: ana zinciri
değiştirmez, `plan` ve `verify` adımlarına ek girdi/doğrulama enjekte eder.

## Tetikleyici
- `atdd.md`'nin "Etkilenen Dosyalar" listesi `.tsx`/`.jsx`/`.vue`/`.css`/`.html`
  içeriyorsa, veya "Benchmark / Başarı Ölçütü" bölümünde bir "Görsel/UI kriteri"
  doluysa.
- Kullanıcı doğrudan "bu sayfayı/component'i tasarla", "UI'ı yeniden şekillendir",
  "arayüzü gözden geçir" derse.
- Tetikleyici yoksa bu dal atlanır, doğrudan standart `plan` çalışır.

## Zincir

```
Discover        (Playwright MCP — gerçek app'i gez, route/component/token haritası)
  ↓
Audit           (better-interface veya tekil better-* — mevcut UI'ın eleştirisi)
  ↓
Design Direction (frontend-design — estetik yön, palette/typography/layout kararı)
  ↓
plan            (standart atdd `plan` skill'i — bu adımın çıktılarını plan.md'ye ekler)
  ↓
test-copilot / code-copilot   (standart zincir, değişmez)
  ↓
Validate        (verify + Playwright multi-viewport screenshot + vision-test)
  ↓
Review          (desktop/tablet/mobile + hover/focus/reduced-motion checklist)
  ↓
Iterate         (sadece kanıtlanan sorun düzeltilir, spekülatif "polish" eklenmez)
  ↓
red-team        (standart zincir, değişmez — Review adımının bulguları girdi olur)
```

## Adımlar

1. **Discover.** Playwright MCP (`mcp__playwright__browser_navigate`,
   `browser_snapshot`) ile uygulamayı gerçek tarayıcıda aç. Route listesini,
   kullanılan component'leri, mevcut design token'ları (renk/spacing/font
   değişkenleri varsa) ve genel layout yapısını çıkar. Bunu `plan.md`'nin
   "Dependencies" bölümüne ek not olarak yaz — `plan` skill'i statik kod
   okumasıyla bunu tam yakalayamaz, render edilmiş sonucu görmek gerekir.
2. **Audit (mevcut UI varsa).** `better-interface` skill'ini çağır (tek
   seferde accessibility/layout/typography/color/UI polish hepsini tarar).
   Kapsam dar ve tek konuya odaklıysa `better-layout`/`better-ui`/
   `better-typography`/`better-colors`/`better-accessibility`'den ilgili
   olanı tek başına çağır. Çıktıyı "bulgu listesi" olarak sakla — sıradaki
   adımda hangisi bu görev kapsamında düzeltilecek, hangisi kapsam dışı
   kalacak (`Kapsam Dışı` bölümüne yaz) kullanıcıyla netleştir.
3. **Design Direction (yeni UI veya reshape ise).** `frontend-design`
   skill'ini çağır — ürünün konusuna/hedef kitlesine göre bilinçli bir
   palette/typography/layout yönü seçtirir, şablon hissi veren varsayılan
   kararları (mor gradient, jenerik kart düzeni vb.) engeller. Sadece
   mevcut bir bug/polish fix'inde bu adımı atla — gereksiz yeniden tasarım
   riski yaratır.
4. **plan.** Standart `plan` skill'i çağrılır; 1-3. adımların çıktıları
   (route/component haritası, audit bulguları, tasarım yönü) `plan.md`'ye
   girdi olarak verilir, tekrar keşfedilmez.
5. **test-copilot / code-copilot.** Standart zincir — bu skill bu iki
   adıma müdahale etmez.
6. **Validate.** `verify` çalışırken (veya hemen sonrasında) Playwright ile
   en az üç viewport'ta (`desktop`/`tablet`/`mobile` — `resize_window`
   preset'leri) ekran görüntüsü al, ardından `vision-test` skill'i bu
   görüntüleri okuyup yapılandırılmış JSON bulgu üretsin (Claude'un kendi
   vision token'ını harcamadan). Console hata/uyarılarını da
   `read_console_messages` ile kontrol et.
7. **Review.** 6. adımın çıktısını şu checklist'e göre değerlendir:
   - Desktop/tablet/mobile'da layout kırılmıyor mu
   - Hover/focus durumları görünür mü (klavye erişilebilirliği dahil)
   - `prefers-reduced-motion` saygı görüyor mu (varsa animasyon)
   - 2. adımdaki audit bulgularından bu görev kapsamındakiler kapandı mı
   Kapanmayan bir madde varsa 8. adıma geç, hepsi tamamsa `red-team`'e geç.
8. **Iterate.** Sadece 7. adımda **kanıtlanmış** (ekran görüntüsü/console
   log ile doğrulanmış) sorunları düzelt — audit'in "olabilir" dediği ama
   ekranda görünmeyen bir öneriyi görev kapsamına ekleme, bu scope creep'tir.
   Düzeltmeden sonra 6. adıma dön (tek tur, sonsuz döngü değil — ikinci
   turda hâlâ kapanmayan bir madde varsa kullanıcıya raporla, kendi başına
   üçüncü tura girme).
9. **Report.** `red-team`'e geçmeden önce kullanıcıya kısa özet: hangi
   viewport'larda test edildi, hangi audit bulguları kapandı, hangileri
   bilinçli olarak kapsam dışı bırakıldı.

## Guardrail (hookify)
Bu zincirin adımlarını atlamayı zorlaştırmak için önerilen `hookify` kuralları
(`hookify:hookify` ile kurulur):
- CSS/layout/component dosyası değişti → görev "tamamlandı" denmeden önce en az
  bir Playwright screenshot alınmış olmalı (Stop event, warn).
- `console.log`/`debugger` eklendi → commit'ten önce uyar.
- `.env`/credential dosyası değişti → uyar/engelle (genel guardrail, frontend'e
  özel değil ama aynı pipeline'da faydalı).

## Kural
- Bu skill kod yazmaz, kendi başına dosya değiştirmez — sadece hangi adımda
  hangi mevcut skill/MCP aracının çağrılacağını sıralar.
- `Design Direction` adımı yalnızca yeni UI/kapsamlı reshape'te zorunludur;
  küçük bug fix'lerinde atlanır (bkz. `fast-track` ile aynı mantık).
- `Iterate` en fazla bir tur otomatik çalışır — ikinci turda kapanmayan
  bulgu varsa kullanıcıya sorulur, sessizce genişletilmez.
- Playwright MCP güvenlik sınırı taşımaz (bkz. Playwright MCP'nin kendi
  uyarısı) — prod hesap/oturum bilgisiyle değil, izole test ortamıyla
  kullan.
