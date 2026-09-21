---
name: project-vault-cv-tailoring
description: >
  Yusuf Cinar'ın kişisel bilgi/proje "kasası" (Notion'da, "Yusuf Cinar -
  Proje Kasası" ana sayfası altında, her proje ayrı alt-sayfa) mevcut
  olduğunda ve yeni bir şirkete özel CV hazırlanırken kullan. KASANIN
  TAMAMINI ASLA OKUMA - çok yer kaplar, bağlamı gereksiz doldurur. Sadece
  o ilana uygun 2-4 projeyi hedefli sorgula. Mevcut en güncel CV
  (cv/main_<company>_<role>.tex, en sonuncusu) her zaman TEMPLATE'tir -
  sıfırdan yazma, ondan tailor et. Triggers: "kasadan CV hazırla",
  "yeni şirket için CV", "kasayı kullan", "template'ten CV çıkar".
---

# Proje Kasası + CV Template Akışı

## Neden bu skill var

Kullanıcının kişisel bilgi/proje geçmişi, GitHub repo analiziyle
zenginleştirilmiş büyük bir Notion "kasası"nda tutuluyor (`Yusuf Cinar -
Proje Kasası` ana sayfası + her proje için ayrı alt-sayfa, bkz.
`manus-notion-bridge` ile üretildi). Bu kasa büyüdükçe **tamamını her CV
hazırlığında okumak bağlamı gereksiz yere dolduracak** - bu yüzden erişim
her zaman **hedefli** olmalı.

## Kural 1: Kasanın tamamını asla okuma

- `notion-fetch` ile SADECE ana sayfanın (`Yusuf Cinar - Proje Kasası`)
  kendisini çek - bu sadece alt-sayfa başlıklarının/linklerinin listesini
  verir, her alt-sayfanın tam içeriğini DEĞİL.
- İlana bakıp hangi 2-4 projenin alakalı olduğuna karar ver (anahtar
  kelime eşleşmesi: ilanın istediği teknoloji/alan ile proje başlığı/kısa
  özeti). Sadece o 2-4 alt-sayfayı `notion-fetch` ile tek tek çek.
- Emin değilsen `notion-search` ile spesifik bir terim ara (örn. "RAG",
  "embedded", "computer vision") - kasanın tamamını listelemek yerine
  ilgili sayfayı bulmanın yolu bu.
- **Asla** "kasadaki her sayfayı sırayla oku" gibi bir döngü kurma.

## Kural 2: Mevcut CV her zaman template'tir

- `cv/` klasöründeki en son yazılmış `main_<company>_<role>.tex` dosyası
  (git log'da veya dosya tarihinde en yeni olan) yapısal referanstır -
  `05-cv-templates.md`'nin zaten söylediği kural bu, burada tekrar
  ediliyor çünkü kasa akışına özel önemi var: kasadan gelen YENİ bir
  proje detayını CV'ye eklerken bile moderncv/banking format, bölüm
  sırası, `\needspace`/2-sayfa bütçesi gibi her şey template'teki gibi
  kalmalı. Sıfırdan yeni bir CV yapısı icat etme.
- Kasadan çekilen proje detayı, `01-candidate-profile.md`'nin yerini
  almaz - o hâlâ "tek gerçek kaynak" (grounding audit `01` +
  `cv/main_example.tex` + `CLAUDE.md` üçlüsüne bakıyor, `05-cv-templates.md`
  Step 3 Factual Grounding Audit). Kasadan gelen bir teknik detay
  (örn. GitHub kodundan çıkan bir mimari kararı) CV'ye eklenecekse,
  önce `01-candidate-profile.md`'ye de yazılmalı ki tutarlılık bozulmasın
  - bu, `apply.md`'nin "write new facts back to the profile" kuralının
  kasa kaynaklı bilgiler için de geçerli olması demek.

## Akış

```
1. Yeni ilan geldiğinde (ATDD/apply akışı zaten çalışıyor).
2. Kasa ana sayfasını çek (sadece başlık listesi).
3. İlanla en alakalı 2-4 alt-sayfayı seç, SADECE onları notion-fetch et.
4. En son CV'yi (cv/main_*.tex, en yeni) oku - template.
5. Kasadan gelen yeni/daha zengin proje detaylarını, varsa
   01-candidate-profile.md'ye de ekleyerek, CV'nin ilgili
   Projects/Experience bölümüne tailor et.
6. Her zamanki compile-and-inspect döngüsü (05-cv-templates.md) değişmez.
```
