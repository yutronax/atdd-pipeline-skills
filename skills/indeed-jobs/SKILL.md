---
name: indeed-jobs
description: Bağlı Indeed MCP araçlarını (search_jobs, get_job_details, get_company_data, get_resume) kullanma referansı. "Indeed'de ara", "bu şirket nasıl", "Indeed özgeçmişimi çek" gibi isteklerde, veya linkedin/kullanici-profili staj-başvuru hattına yeni ilan beslerken kullan.
---

# Indeed MCP Araçları — Kullanım Referansı

Sunucu id'si oturumdan oturuma değişebilir (örn. `mcp__4717e47d-...__search_jobs`).
Tool adı sabit kalıyor (`search_jobs`, `get_job_details`, `get_company_data`,
`get_resume`) — çağırmadan önce gerekirse `ToolSearch` ile güncel id'yi bul
(`query: "select:search_jobs,get_job_details,get_company_data,get_resume"`).

## Araçlar

| Araç | Zorunlu parametreler | Ne döner | Ne zaman kullan |
|---|---|---|---|
| `search_jobs` | `search` (başlık/anahtar kelime), `location` (şehir veya `"remote"`), `country_code` (ISO 3166 iki harf, örn. `TR`, `US`) | Markdown liste — her ilan başlığı apply linkine gömülü | Yeni ilan taraması, "X pozisyonu ara" |
| `get_job_details` | `job_id` | Tek ilanın tam açıklaması, maaş, başvuru linki | `search_jobs` sonucundan seçilen bir ilanı derinlemesine incelemek |
| `get_company_data` | `companyName`, `language`, `location` (`{country, usState, usStateCode, usCity}` — bilinmiyorsa hepsi `null`), `knowledgeCategories` (`{metadata, ratings, salaries}` — emin değilsen üçü de `true`) | Şirket kültürü, maaş, CEO onayı, çalışan yorumları | Başvurudan önce şirketi değerlendirmek; kullanıcı "bu şirket nasıl" derse |
| `get_resume` | (parametresiz) | Indeed'e bağlı özgeçmiş verisi | Diğer araçlardan önce arka plan bilgisi almak, veya `kullanici-profili/profile.md` ile karşılaştırmak |

İsteğe bağlı: `search_jobs`'ta `job_type` — `fulltime`/`parttime`/`contract`/`internship`/`temporary`.

## Kurallar (tool açıklamalarından, atlanmaz)

- **Apply linkini her zaman göster** — her ilan başlığına link gömülü olmalı, kullanıcı tıklayıp başvurabilsin.
- **URL'leri bozma** — query parametrelerini (özellikle `jk=` job id) kırpma/kısaltma.
- `get_company_data`: tek seferde tek şirket, tek job title. Kullanıcı birden fazla şirket sorarsa ayrı ayrı çağır.
- Bu araçlar salt-okunur arama/bilgi araçlarıdır — başvuru gönderme (submit/apply) aracı yok. Gerçek başvuru her zaman kullanıcının kendisi tarafından, apply linkine tıklayarak yapılır.

## `linkedin` / `kullanici-profili` hattına entegrasyon

Bu iki proje zaten bir Indeed-kaynaklı ilan şemasına sahip
(`linkedin/reports/report_*.json`, JobsPipe agregatörü üzerinden dolduruluyor):

```json
{"id": "...", "title": "...", "company": "...", "location": "...",
 "url": "https://www.indeed.com/viewjob?jk=...", "description": "...",
 "remote_type": null, "posted_at": "..."}
```

`job_id` bu JSON'daki `url`'nin `jk=` parametresiyle aynıdır — yani
`search_jobs`/`get_job_details` sonuçlarından bu şemaya uygun bir JSON objesi
elle kurup `kullanici-profili/cv_creator.py --job-json <dosya> --job-index 0`
ile doğrudan besleyebilirsin (kodda değişiklik gerekmez, `main.py` zaten bu
JSON'u okuyor — bkz. `kullanici-profili/cv_creator.py:113-119`).

Akış:
1. `search_jobs` ile tara → uygun ilanı seç.
2. `get_job_details(job_id)` ile tam açıklamayı al.
3. İstersen `get_company_data(companyName=...)` ile şirketi değerlendir.
4. Yukarıdaki şemayla bir JSON dosyası yaz (`reports/` altına veya scratch'e).
5. `cv_creator.py --job-json ... --output kullanici-profili/tailored_cv_<sirket>.md`
   ile ilana özel CV üret.
6. Mail taslağı için mevcut örneği şablon al:
   `kagen_robotics_basvuru_mail.txt` — `Kime:` / `Konu:` / gövde
   (pozisyon + 2-3 ilgili proje + kapanış + iletişim bilgileri) formatı.

## Sık kullanım örnekleri

- "Indeed'de Ankara'da robotik stajı ara" → `search_jobs(search="robotik stajyeri", location="Ankara", country_code="TR", job_type="internship")`
- "Şu ilanın detayını göster" → önce `search_jobs`'tan `job_id` çıkar, `get_job_details(job_id)`
- "Bu şirkette çalışmak nasılmış" → `get_company_data(companyName=..., knowledgeCategories={metadata:true, ratings:true, salaries:true}, location={country:"TR", usState:null, usStateCode:null, usCity:null}, language="tr")`
- "Indeed özgeçmişimi çek, profile.md ile karşılaştır" → `get_resume()` sonra `kullanici-profili/profile.md` ile diffle
