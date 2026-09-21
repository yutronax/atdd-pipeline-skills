# vps-deploy — Mavi Lojistik VPS'e Deploy (bilinen sorunları tekrarlamadan)

Bu skill, `<PROJECT_REPO>` reposunu canlı VPS'e (`<VPS_IP>`)
deploy ederken defalarca yaşanan 4 gerçek sorunu bir daha yaşamamak için
yazıldı. Her adım, canlı olarak tespit edilmiş bir hatayı önlüyor —
sırayı atlama.

## Ön Koşul
`main`'e merge edilmiş bir commit olmalı (bu skill merge ETMEZ, sadece
deploy eder). Merge için ayrı: `gh pr create`/`gh pr merge` (bkz. aşağıda
Adım 0 — GITHUB_TOKEN sorunu).

## Bilinen Ortam Kısıtları (asla varsayma, her seferinde doğrula)

1. **Bu CLI ajanı ortamından VPS'e SSH erişimi YOKTUR.** `<VPS_IP_OLD>`
   (bazı eski config dosyalarında geçen adres) bu sandboxdan hiç yanıt
   vermiyor (timeout, tüm portlar). `<VPS_IP>:2222` portu açık ama
   bu ortamda geçerli bir private key yok → `Permission denied
   (publickey)`. **Sonuç: tüm SSH komutlarını KULLANICIYA ver, kendin
   çalıştırmayı deneme** — kullanıcı kendi PowerShell'inden çalıştırıp
   çıktıyı yapıştırır.
2. **Gerçek VPS adresi `<VPS_IP>`, port `<SSH_PORT>`, kullanıcı `<VPS_USER>`.**
   Panel HTTP portu `8080`. Proje dizini `<PROJECT_PATH>`.
3. **PM2 process isimleri** (`ecosystem.config.js`'den): `<PM2_SERVER_PROCESS>`,
   `<PM2_ADMIN_PROCESS>`, `mavi-baileys-bridge`. Restart ederken isimle hedefle,
   `pm2 restart all` KULLANMA (baileys-bridge'i gereksiz yere restart eder).

## Adım 0 — Merge (deploy'dan ÖNCE)

`gh auth status` bu makinede genelde `GITHUB_TOKEN` env var'ı ile geliyor
ve bu token'ın `createPullRequest` için scope'u YOK (`GraphQL: ... requires
... ['public_repo']`). Çözüm: `GITHUB_TOKEN`'ı o tek komut için devre dışı
bırak, `keyring` hesabı (`repo` scope'lu) devreye girsin:

```bash
env -u GITHUB_TOKEN gh pr create --base main --head <branch> --title "..." --body "..."
env -u GITHUB_TOKEN gh pr merge <PR-numarası> --merge --delete-branch=false
```

## Adım 1 — VPS'in local `main`'inin durumunu ÖNCE kontrol et

**ASLA doğrudan `git pull` çalıştırma.** VPS'in local `main`'i, önceki
deploy'larda oluşmuş yerel merge commit'leri (`Merge remote-tracking
branch 'origin/main'`) yüzünden neredeyse HER SEFERİNDE origin'den
diverge etmiş oluyor — bu, gerçek bir kod çakışması değil, sadece geçmiş
gürültüsü. `git pull` bu durumda ya "Diverging branches can't be
fast-forwarded" hatası verir ya da (yeni git sürümlerinde) "Need to
specify how to reconcile divergent branches" ile tamamen durur.

Kullanıcıya verilecek komut (PowerShell'den, tek satır — bkz. aşağıda
"PowerShell Tırnak Kuralı"):

```bash
ssh -p <SSH_PORT> <VPS_USER>@<VPS_IP> "cd <PROJECT_PATH> && git fetch origin main && git merge origin/main --no-ff -m 'Merge origin main deploy sync' && git log --oneline -3"
```

`git merge --no-ff` diverge durumunu sorunsuz çözer (data dosyalarında
commit'lenmemiş yerel değişiklikler varsa bile — PR'ın kendisi `data/*.json`
dosyalarına dokunmuyorsa çakışma olmaz, dokunuyorsa önce kullanıcıyı
uyar).

## Adım 2 — PM2 restart + canlı doğrulama (TEK komutta, VPS'in KENDİ içinden test et)

Dış dünyadan `curl http://<VPS_IP>:8080` testi YANILTICI olabilir
(firewall/network farklı davranabilir) — her zaman VPS'in kendi
`127.0.0.1`'inden test et:

```bash
ssh -p <SSH_PORT> <VPS_USER>@<VPS_IP> "cd <PROJECT_PATH> && pm2 restart <PM2_SERVER_PROCESS> <PM2_ADMIN_PROCESS> && sleep 2 && pm2 status && curl -s -o /dev/null -w localcode=%{http_code}\n http://127.0.0.1:8080"
```

`localcode=200` → başarılı. `localcode=000` → Adım 3'e geç.

## Adım 3 — `localcode=000` ise: `.venv` kontrolü (canlı olarak tekrarlayan kök neden)

PM2 "online" gösterse bile gerçek süreç ayakta olmayabilir — belirti:
`pid N/A`, `mem 0b`, `pm2 logs` tamamen boş, restart sayacı (`↺`) çok
yüksek (100+). Kök neden: `ecosystem.config.js`'nin beklediği
`.venv/bin/python3` VPS'te YOK (sadece sistem Python'u var).

Teşhis:
```bash
ssh -p <SSH_PORT> <VPS_USER>@<VPS_IP> "cd <PROJECT_PATH> && ls -la .venv/bin/ 2>&1; which python3 && python3 --version"
```

`.venv/bin/` yoksa, düzelt:
```bash
ssh -p <SSH_PORT> <VPS_USER>@<VPS_IP> "cd <PROJECT_PATH> && python3 -m venv .venv && ./.venv/bin/pip install --no-cache-dir --upgrade pip && ./.venv/bin/pip install --no-cache-dir -r requirements.txt"
```
Sonra Adım 2'yi tekrarla.

## Adım 4 — `ecosystem.config.js`'in `env` bloğuna yeni bir değişken eklendiyse (canlı olarak tekrarlayan kök neden, 2026-09-12)

**`pm2 restart <isim>` (dosya belirtmeden), `--update-env` ile bile,
`ecosystem.config.js`'i YENİDEN OKUMAZ** — PM2 sadece daemon'ın kendi
internal process kaydındaki (önceden kaydedilmiş) env'i tekrar uygular.
Bu, `baileys-sidecar-webhook-secret-header` görevinde ~45 dakikalık bir
teşhis sürecine yol açtı: `ecosystem.config.js`'e `WEBHOOK_SHARED_SECRET`
eklendi, `pm2 restart mavi-baileys-bridge --update-env` defalarca
çalıştırıldı, ama process HİÇBİR ZAMAN yeni değişkeni görmedi (`pm2 logs`
sürekli "WEBHOOK_SHARED_SECRET is not defined" basmaya devam etti) —
dosya doğruydu, `pm2 describe` de doğru `script path`/`cwd` gösteriyordu,
ama env yine de eski kalıyordu.

**Kesin çözüm — `restart` değil, `delete` + dosyadan `start`:**
```bash
ssh -p <SSH_PORT> <VPS_USER>@<VPS_IP> "cd <PROJECT_PATH> && pm2 delete <process-adi> && pm2 start ecosystem.config.js --only <process-adi>"
```
Bu, PM2'ye dosyayı gerçekten SIFIRDAN okutur. Sadece `--update-env`'e
güvenip "sorun VPS'te değil, .env eksik" diye tekrar tekrar aynı yanlış
teşhise dönmek yerine, `env` bloğu her değiştiğinde doğrudan bu deseni
kullan — `pm2 restart` sadece kod dosyası (`bridge.js`/`vps_main.py` vb.)
değiştiğinde yeterlidir, `ecosystem.config.js`'in kendisi değiştiğinde
YETERSİZDİR.

**Doğrulama ipucu:** "gerçekten düzeldi mi" sorusuna `pm2 logs`'un "son N
satırı" ile cevap arama — bu, dosyada duran ESKİ log satırlarını da
gösterebilir (yanıltıcı). Bunun yerine güncel sunucu saatiyle karşılaştır:
```bash
ssh -p <SSH_PORT> <VPS_USER>@<VPS_IP> "date && grep -c '<hata-metni>' <log-dosyasi> && grep '<hata-metni>' <log-dosyasi> | tail -3"
```
Sayı restart'tan sonra hiç ARTMIYORSA (aynı sayı, aynı son zaman damgası)
ve güncel saat o zaman damgasından yeterince (birkaç dakika) ileriyse,
düzelme gerçek — sadece "log'da görünmedi" değil.

## PowerShell Tırnak Kuralı (canlı olarak defalarca kırıldı)

Kullanıcı Windows PowerShell 5.1 kullanıyor. Komutu HER ZAMAN şu formatta
ver — **dış çift tırnak, iç TEK tırnak**, backslash-escaped çift tırnak
(`\"..\"`) KULLANMA (PowerShell bunu parçalayıp "is not recognized as a
cmdlet", "Diverging..." veya `sed: unterminated 's' command` gibi
alakasız hatalara yol açıyor):

```
ssh -p <SSH_PORT> <VPS_USER>@<VPS_IP> "cd <PROJECT_PATH> && git merge origin/main --no-ff -m 'tek tirnakli mesaj' && ..."
```

`&&` zincirleme SSH komutunun İÇİNDE (uzak bash'te) çalıştığı için sorun
değil — sorun sadece PowerShell'in KENDİ üst seviyesinde `&&`/escape
karışıklığı. Her zaman TEK SSH komutu ver, birden fazla ayrı PowerShell
komutu istemekten kaçın.

**Uzak taraftaki bir dosyada (örn. `ecosystem.config.js`) literal çift
tırnak İÇEREN bir metin araman/değiştirmen gerekiyorsa** (`sed`'in `s#"..."#..."#'`
gibi kalıpları), `\"` denemekten VAZGEÇ — PowerShell bunu güvenilir şekilde
taşımıyor (`baileys-sidecar-webhook-secret-header` deploy'unda `sed:
unterminated 's' command` hatasıyla defalarca kırıldı). Bunun yerine
düzenleme mantığını bir Python script'i olarak yaz, `base64 -w0` ile
kodla, ve TEK komutta gönder — bu, hiç tırnak/özel karakter içermediği
için PowerShell'den kayıpsız geçer:

```bash
# (önce Python script'i base64'e çevir, sonra:)
ssh -p <SSH_PORT> <VPS_USER>@<VPS_IP> "cd <PROJECT_PATH> && echo <base64-blok> | base64 -d | python3 - && <doğrulama-komutu>"
```

Script içinde `old`/`new` string'leri `str.replace()` ile eşleştir, eşleşme
bulunamazsa (`PATTERN_NOT_FOUND`) veya zaten uygulanmışsa (`ALREADY_PATCHED`)
açıkça çıktı ver — sessiz başarısızlık olmasın.

## Kontrol Listesi (deploy'u "bitti" demeden önce)
- [ ] PR merge edildi mi (`gh pr view <n> --json state` → `MERGED`)
- [ ] VPS'in local `main`'i origin ile senkron mu (`git log --oneline -3` origin ile eşleşiyor mu)
- [ ] PM2 restart edilen process'ler `online` VE gerçekten `pid` ataması var mı (N/A değil)
- [ ] `curl 127.0.0.1:8080` VPS'in kendi içinden 200 dönüyor mu
- [ ] Değişen kodun gerçekten VPS'e ulaştığı doğrulandı mı (ör. `grep -c <yeni-fonksiyon-adı> <dosya>`)
