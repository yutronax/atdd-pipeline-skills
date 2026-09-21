---
name: ai-job-search-outreach
description: >
  Yusuf Cinar'ın soğuk-başvuru/staj arama akışının uçtan uca orkestrasyon
  referansı - şirket bulmaktan CV hazırlamaya, mail göndermeye ve
  kayıt tutmaya kadar hangi adımda hangi araç/skill kullanılır, hangi
  sırayla. Bu skill'in kendisi iş yapmaz, üç alt-skill'i (manus-notion-bridge,
  project-vault-cv-tailoring, job-search-calendar-gmail) doğru sırada
  bağlar. "yeni şirket bul", "staj başvurusu hazırla", "şirkete mail at",
  "kasadan CV çıkar" gibi isteklerde bu akışı takip et.

  2026-09-03/04'te ai-job-search reposu (scratchpad/ai-job-search) ile
  canlı kurulup test edildi: Vendilabs (LaTeX CV), sonra Insider
  One/SESTEK/Vispera/Teknopar/MİA Teknoloji/Softalya/Doruk Otomasyon/
  Baksoft Arge (docx CV) - 4 gerçek email gönderildi, 1 bounce yaşandı,
  3'ü form/portal gerektirdiği için email atılmadı. Bu skill o oturumda
  öğrenilen her kuralı içeriyor.
---

# İş Arama Soğuk-Başvuru Akışı - Orkestrasyon

## Genel akış (sırayla)

```
1. ŞİRKET/İLAN BUL           → manus-notion-bridge
2. KANAL BELİRLE              → bu dosya, "Kanal Kararı" bölümü
3. EMAİL ADRESİ DOĞRULA        → bu dosya, "Email Doğrulama" bölümü
4. CV'Yİ ŞİRKETE ÖZEL HAZIRLA  → project-vault-cv-tailoring + bu dosya
5. MAİL METNİ YAZ              → bu dosya, "Mail İçeriği" bölümü
6. GÖNDER (onaylı)             → job-search-calendar-gmail
7. BOUNCE KONTROLÜ              → bu dosya, "Bounce Kontrolü" bölümü
8. KAYDET (takvim + tracker)   → job-search-calendar-gmail + bu dosya
```

## 1. Şirket/ilan bulma

`manus-notion-bridge` skill'ini çağır. Manus prompt'una şehir/sektör
kısıtını net yaz (ör. "SADECE Antalya"), sonucu Notion'a yazmasını
iste. Büyük kapsamlı (15-20+ şirket, derin doğrulama) görevler
`mode: quality` ile uzun sürüp takılabiliyor (canlı olay) - 8-10
şirketlik parçalara bölüp `mode: speed` kullan.

## 2. Kanal Kararı — ZORUNLU, atlanamaz

**Kural: Şirketin kendi başvuru sistemi (form/portal/ATS) varsa MAİL ATMA.**
Bu, kullanıcının açık talimatı ("başvurusu varsa mail atma") ve aynı
zamanda mantıklı: bir formu/portalı olan şirkete email atmak muhtemelen
görülmeyecek/yanlış kanaldır.

Her şirket için Manus'un veya kendi WebFetch'inin bulduğu kariyer
sayfasını kontrol et:
- **Somut açık ilan var** (örn. "4. Sınıf Öğrencisi için Staj" gibi
  isimlendirilmiş bir pozisyon) → o ilanın kendi başvuru linkinden
  başvurulmalı, email atma.
- **Harici bir kariyer portalına yönlendiriyor** (`careers-page.com`,
  Kariyer.net, LinkedIn Jobs vb.) → o portaldan başvurulmalı, email
  atma.
- **Sitede gömülü bir başvuru formu var** ("Genel Başvuru", "Staj
  Başvurusu" modalı) → formu doldurmalı, email atma.
- **Hiçbir form/portal yok, sadece bir email adresi var** → soğuk
  email atılabilir, Adım 3'e geç.

Bu kontrolü atlayıp doğrudan email atma - üç gerçek şirkette (Vispera,
Teknopar, MİA Teknoloji) tam bu nedenle email değil form/portal
kullanılması gerektiği tespit edildi.

Form/portal gereken şirketler için: kullanıcıya linki ver, `job_search_tracker.csv`'ye
`channel: form` veya `portal`, `status: drafted` olarak not düş - mail
atmadığını unutma, ama takip edilmesi gerektiğini de kaybetme.

**İstisna (yalnızca kullanıcı onayıyla):** Kullanıcı özellikle isterse,
form/portalı olan bir şirkete de ek bir kanal olarak email atılabilir -
ama email formdan hiç bahsetmemeli, sıradan bir soğuk başvuru gibi
yazılmalı (canlı olay: Asyasoft ve Görsentam'ın Kariyer.net/kendi
kariyer sayfası kanalı vardı, kullanıcı "form olduğunu belirtmeden mail
at" dedi, ikisine de gönderildi). Bu varsayılan davranış DEĞİL - her
seferinde ayrı sorulmalı, otomatik genelleme yapılmamalı.

## 3. Email Doğrulama — ZORUNLU, atlanamaz

Manus'un bulduğu bir email adresine **asla körü körüne güvenme**.
Canlı olay: Manus "turkey@useinsider.com" adresini verdi, gerçekte o
grup mevcut değildi ve mail bounce oldu (Google mailer-daemon hatası:
"the group you tried to contact (turkey) may not exist").

Göndermeden önce: `WebFetch` ile şirketin kendi kariyer sayfasını çek,
"bu adres bu sayfada gerçekten geçiyor mu?" diye doğrula. Sayfada
görünmeyen bir adrese gönderme.

## 4. CV'yi Şirkete Özel Hazırlama

`project-vault-cv-tailoring` skill'inin kurallarına göre çalış (kasanın
tamamını okuma, ilgili 2-4 kaynağı hedefli çek). Somut uygulama:

- **Master template:** `scratchpad/ai-job-search/cv_docx/main_template.docx`
  (kullanıcının kendi verdiği fotoğraflı/tek-sütun format - moderncv/LaTeX
  DEĞİL, kullanıcı LaTeX formatını beğenmedi).
- **Tailor script:** `scripts/tailor_cv_ozet.py --master <template> --company <key>`
  Sadece ÖZET paragrafını ve (varsa) adres şehrini değiştirir, DENEYİM/
  PROJELER/YETENEKLER/EĞİTİM aynı kalır - moderncv'nin "en önemli
  özelleştirme profile statement'tır" ilkesiyle aynı mantık.
  - Yeni bir şirket eklerken `OZET_VARIANTS` sözlüğüne bir giriş ekle
    (o şirketin alanına göre hangi proje/deneyim öne çıkarılacak),
    gerekirse `ADDRESS_OVERRIDES`'a da ekle (kullanıcının gerçekten
    ikamet ettiği şehir - Elazığ/İstanbul/Antalya; olmayan bir şehri
    asla yazma).
  - Çıktı dosya adı **gerçek bir CV adı olmalı**: `Yusuf_Cinar_CV_<Şirket>.docx`
    - `main_<key>.docx` gibi internal/teknik bir isim DEĞİL (kullanıcı
      bunu açıkça düzeltti). Script zaten varsayılan olarak doğru
      isimlendiriyor, `--out` ile ezme.
- **PDF'e çevir:** `"C:\Program Files\LibreOffice\program\soffice.exe" --headless --convert-to pdf --outdir <dir> <docx>`
- **Fabrikasyon yasak:** ÖZET'te vurgulanan proje/deneyim gerçekten
  `01-candidate-profile.md`'de olmalı - yeni bir iddia uydurma, sadece
  var olanın hangisinin öne çıkacağını seç.
- **CV dili:** Türkçe (kullanıcı kararı, `CLAUDE.md` Identity bölümünde
  kayıtlı - İngilizce değil).
- **OBSS staj tarihi gibi profil değişiklikleri:** Önce `CLAUDE.md` +
  `01-candidate-profile.md`'yi güncelle, SONRA `main_template.docx`'i
  güncelle, SONRA tüm ilgili `Yusuf_Cinar_CV_*.docx` dosyalarını
  template'ten yeniden üret - tek bir yerde düzeltip diğerlerini
  unutma.

### Bilinen tuzak: docx run metnini değiştirirken boşluk/tab kaybı

`python-docx` ile bir run'ın `.text`'ini komple değiştirirken, o run'ın
başında/sonunda görünmez bir tab/boşluk karakteri varsa (örn. başlık ile
tarih arasındaki hizalama tab'ı) onu da silersin - "Yapay Zeka
StajyeriTem 2026" gibi bitişik, bozuk bir çıktı alırsın. Canlı olay:
tam bunu yaşadık, düzeltilmiş metne `\t` önekini geri eklemek gerekti.
**Her docx düzenlemesinden sonra PDF'e çevirip Read tool ile görsel
kontrol et** - moderncv'nin "compile-and-inspect loop" kuralının docx
karşılığı bu.

## 5. Mail İçeriği

`03-writing-style.md` kuralları geçerli (em-dash yok, klişe yok,
doğrulanmamış şirket iddiası yok). Format: kısa selam → şirkete neden
yazdığın (1-2 cümle, şirketin gerçek alanına atıf) → 1-2 somut proje/
deneyim (gerçek metrik) → kısa kapanış + CV ekte notu + imza (isim,
email, telefon, LinkedIn, GitHub).

## 6. Gönderim

`job-search-calendar-gmail` skill'inin kuralı aynen geçerli: **ek
varsa `scripts/send_application_email.py` kullan, Gmail MCP
`create_draft`/`send_message` değil** (dosya eki MCP üzerinden
Claude'un bağlamından geçmek zorunda, yavaş ve hataya açık - canlı
olay: 66KB'lık bir PDF için 4 parçaya bölünmüş Read gerekti).

`GMAIL_ADDRESS`/`GMAIL_APP_PASSWORD` ortam değişkenleri zaten
kullanıcının mevcut PowerShell oturumunda tanımlı değilse, komuttan
hemen önce `$env:` ile o çağrı için ayarla (registry'ye `setx` ile
yazmak açık oturumları etkilemiyor, yeni terminal gerektiriyor - canlı
olay).

**Her zaman**: önce `--dry-run` ile doğrula (ilk kurulumda), gerçek
gönderim komutu çalıştırılmadan önce kullanıcıdan açık onay al - script
var olması onay kuralını gevşetmiyor.

## 7. Bounce Kontrolü — ZORUNLU, atlanamaz

Gönderdikten hemen sonra (aynı turda, birkaç saniye içinde bounce
gelebiliyor - canlı olay: Insider One bounce'ı gönderimden ~25 saniye
sonra geldi):

```
mcp__<gmail>__search_threads(query="<sirket_anahtar_kelimesi> newer_than:1d")
```

`mailer-daemon` gönderen veya "Delivery Status Notification (Failure)"
konu satırı görürsen bounce olmuş demektir - kullanıcıya hemen söyle,
`job_search_tracker.csv`'ye `status: no response`, notlara bounce
sebebini yaz, o kanalı "ölü" olarak işaretle, tekrar deneme.

## 8. Kayıt (Calendar + Tracker)

- **Calendar:** `job-search-calendar-gmail` kuralına göre onaysız,
  otomatik. Format: `<Şirket> - <Rol/Staj> - Başvuruldu`, tüm-gün event,
  `description`'da alıcı email + CV dosya adı + kısa not (bounce olduysa
  onu da).
- **Tracker (`job_search_tracker.csv`):** `/apply` Step 6b'nin şemasını
  kullan (`date,company,sector,role,role_type,channel,status,
  contact_person,fit_rating,notes,cv_file,cover_letter_file,source,deadline`).
  `channel` alanı `email`/`form`/`portal` olarak doğru kanalı yansıtsın
  (Adım 2'nin kararı). Bounce olan satırlar `status: no response` +
  notlarda açık bounce sebebi.

## Kısayol Referansı — Dosya/Script Konumları

| Ne | Nerede |
|---|---|
| Master CV template | `scratchpad/ai-job-search/cv_docx/main_template.docx` |
| Şirkete özel CV üretici | `scripts/tailor_cv_ozet.py` |
| Mail gönderici (ek destekli) | `scripts/send_application_email.py` |
| Aday profili (tek gerçek kaynak) | `.claude/skills/job-application-assistant/01-candidate-profile.md` |
| Hedef sektör/şehir | `CLAUDE.md` Target Sectors bölümü |
| Başvuru kaydı | `job_search_tracker.csv` |
| Şirket araştırma kasası | Notion "Yusuf Cinar - Proje Kasası" + Manus'un yazdığı şirket listesi sayfaları |
