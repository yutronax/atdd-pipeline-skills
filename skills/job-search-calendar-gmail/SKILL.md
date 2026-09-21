---
name: job-search-calendar-gmail
description: >
  ai-job-search iş arama akışında Google Calendar ve Gmail MCP araçlarını
  kullanırken izlenecek kurallar. Calendar SADECE kayıt/hatırlatma amaçlı
  (ne zaman hangi şirkete başvuruldu, mülakat tarihi) - otomatik ve
  onaysız kullanılabilir. Gmail ise gerçek insanlara giden mesajlar
  ürettiği için HER ZAMAN gönderim öncesi kullanıcı onayı gerektirir,
  istisnasız. Triggers: "takvime ekle", "calendar'a işle", "mail
  taslağı yaz", "mail gönder", "başvuru hatırlatıcısı kur".
---

# İş Arama - Calendar ve Gmail Kullanım Kuralları

Test edildi ve doğrulandı (2026-09-03): her iki MCP bağlantısı da çalışıyor.
- Calendar: `mcp__0329dda3-dccd-4909-a77a-39a560cf2ba2__*` (ana takvim:
  `<YOUR_EMAIL>`, Europe/Istanbul saat dilimi)
- Gmail: `mcp__ddc1b4f0-dee3-4593-8df9-ea8fcd5bc382__*` (443 sent, 18908
  inbox - gerçek, aktif kullanılan hesap)

Araç isimleri her oturumda farklı bir hash-önekiyle gelebilir (deferred
tool listesinde görünür) - **isim eşleşmesi için önce `ToolSearch` ile
"calendar" veya "gmail"/mail anahtar kelimesiyle ara**, sabit bir ID'ye
güvenme.

## Calendar - Kayıt amaçlı, düşük risk, onay gerektirmez

**Ne için kullanılır:** `job_search_tracker.csv`'deki başvuru/mülakat
olaylarını görsel/hatırlatıcı olarak takvime yansıtmak. Bu **ek bir kayıt**
katmanıdır, tracker CSV'nin yerini almaz - ikisi birbirini tekrar eder,
biri diğerinin editörü değildir.

**Ne zaman otomatik (onaysız) event oluşturulur:**
- Bir başvuru `drafted` -> `applied` durumuna geçtiğinde (`/outcome` veya
  kullanıcı "başvurdum" dediğinde): o günün tarihinde, tüm-gün bir event.
  `create_event` çağır, onay isteme - bu geri dönüşü kolay, kişisel bir
  kayıt işlemi (Explicit-permission listesindeki "gönderim/yayın/silme"
  kategorisine girmiyor, sadece kullanıcının kendi takvimine kendi
  hakkında not düşme).
- Mülakat tarihi netleştiğinde (`/interview` veya kullanıcıdan gelen bilgi):
  o saatte bir event, hatırlatıcı (`reminders`) ile.

**Event formatı:**
```
summary: "[Şirket] - [Rol] - Başvuruldu"  (veya "Mülakat: [Aşama]")
description: kaynak URL + kısa not (fit skoru, portal)
```

**Silme/değiştirme her zaman onay ister** (geri dönüşü zor kategori) -
`delete_event`/`update_event` çağırmadan önce kullanıcıya söyle.

## Gmail - Gönderim her zaman onay gerektirir, istisnasız

**Bu kural pazarlıksız ve sistem-genelinde zaten var** (bkz. üst
seviye güvenlik kuralları: "Sending any message on the user's behalf"
Explicit-permission kategorisinde). Bu skill onu iş arama bağlamına
uyguluyor, gevşetmiyor.

**Akış:**
1. Kullanıcı "şu şirkete mail at" dediğinde, önce `create_draft` ile
   TASLAK oluştur (gönderilmez, Gmail Drafts'a düşer). Bu adım onay
   gerektirmez çünkü hiçbir şey gitmiyor.
2. Taslağın tam metnini (alıcı, konu, gövde) kullanıcıya göster.
3. Kullanıcı açıkça onaylamadan (`"gönder"`, `"evet"` gibi net bir yanıt)
   **asla** `send_message` çağırma. "Taslağı hazırladım, gönderiyorum"
   gibi bir cümle kurup onay beklemeden göndermek de yasak - onay
   *ayrı bir tur* olarak gelmeli.
4. Aynı taslağı ikinci kez göndermek istersen (örn. kullanıcı "bunu X'e
   de gönder" derse) bu da **yeni bir onay** ister - önceki onay tek
   seferliktir, genellemez (üst seviye kural: "Permission is per-action").
5. **Toplu gönderim yasak.** Birden fazla şirkete aynı anda, tek onayla
   mail atmak (`manus-notion-bridge`'in bulduğu 9 şirket gibi bir listeye
   hepsine "gönder" denip otomatik döngüyle yollamak) bu repo'nun
   `/apply` felsefesine de aykırı (bkz. ai-job-search deposu: "tek
   seferde bir başvuru", her uygulama insan onaylı). Her alıcı için ayrı
   taslak, ayrı gösterim, ayrı onay.

**Ek (attachment) gönderiliyorsa `create_draft`/`send_message` DEĞİL,
`scripts/send_application_email.py` kullan.** Gmail MCP tool'larına dosya
eki vermek, dosyanın tam base64 içeriğinin Read tool üzerinden Claude'un
konuşma bağlamından geçmesini gerektiriyor - küçük olmayan bir PDF için bu
yavaş, token-israflı ve (dosya birden fazla parçaya bölünüp okunduğunda)
gereksiz karmaşık. Script dosyayı diskten doğrudan okuyup SMTP ile
gönderiyor, hiçbir dosya içeriği Claude'un bağlamına girmiyor. Kurulum:
`GMAIL_ADDRESS`/`GMAIL_APP_PASSWORD` ortam değişkenleri (App Password,
normal şifre değil - https://myaccount.google.com/apppasswords).
Kullanım: `--dry-run` ile önce içeriği doğrula, sonra gerçek gönderim -
**gerçek gönderim komutu (dry-run olmadan) çalıştırılmadan önce yine her
zaman kullanıcı onayı gerekir**, script'in var olması onay kuralını
gevşetmez.

**İçerik kalitesi:** Soğuk-başvuru mailleri `03-writing-style.md`'deki
kurallara tabi (em-dash yok, klişe yok, doğrulanmamış şirket iddiası yok,
her iddia CV/profildeki gerçek bir şeye dayanmalı). Kısa tut (3-5
paragraf), CV/cover letter'ı ek olarak bahset ama bu skill dosya
eklemeyi otomatik yapmaz - kullanıcı ekleri kendi gönderirken elle
ekler ya da ayrı olarak istenir.

## Özet - Karar Tablosu

| Eylem | Onay gerekir mi? | Araç |
|---|---|---|
| Başvuru yapıldığında takvime not düşmek | Hayır | `create_event` |
| Mülakat tarihini takvime eklemek | Hayır | `create_event` |
| Takvim event'ini silmek/değiştirmek | Evet | `delete_event`/`update_event` |
| Soğuk-başvuru mail taslağı yazmak | Hayır (taslak, gönderilmiyor) | `create_draft` |
| O taslağı göndermek | **Evet, her seferinde** | `send_message` |
| Birden fazla kişiye toplu gönderim | **Asla otomatik yapma** | - |
