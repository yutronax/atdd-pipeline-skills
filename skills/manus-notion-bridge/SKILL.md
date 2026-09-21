---
name: manus-notion-bridge
description: >
  Manus'a (mcp__manus-mcp__create_task) bir araştırma/arama görevi devrederken
  ve sonucunu almak isterken kullan. Sonucu Manus'un kendi web arayüzünden
  (Browser pane) okumaya ÇALIŞMA — bu kanal defalarca kanıtlanmış şekilde
  güvenilmez (oturum düşüyor, sekmeler senkron olmuyor, sayfa virtualization
  yüzünden içerik render olmuyor). Bunun yerine Manus'a sonucu kendi Notion
  entegrasyonu üzerinden bir Notion sayfasına yazmasını söyle, sonra o sayfayı
  Notion MCP (notion-search / notion-fetch) ile oku. Triggers: "manus'a sor",
  "manus'ta araştır", "derin arama yaptır", "manus görevi", "/manus-research".
---

# Manus → Notion Köprüsü

## Neden bu skill var

2026-09-03'te canlı olarak şu sorunlar yaşandı ve tekrarlanmaması için buraya
kayıt edildi:

1. **Browser pane (izole tarayıcı) ile takip güvenilmez.** Kullanıcının
   gerçek Edge'i ile Claude'un Browser pane'i tamamen ayrı oturumlardır,
   birbirinden habersizdir. Kullanıcı "bitti" görse bile Browser pane'de
   hiç görünmeyebilir (defalarca 4/4 veya 5/5 adımda takılı kaldı, içerik
   hiç render olmadı — muhtemelen sohbet geçmişi virtualize render ediliyor
   ve scroll edilmeden DOM'a gelmiyor).
2. **Browser pane oturumu sessizce düşebilir** (login sayfasına döner),
   bu durumda navigate/get_page_text komutları "denied" hatası verir ve
   yeniden giriş kullanıcıdan istenmelidir — ki bu da her yeni sohbette
   tekrarlanan bir sürtünme noktasıdır (Claude Code oturumlar arası
   Browser pane state'i kalıcı tutmaz).
3. **Notion köprüsü kanıtlanmış şekilde çalışıyor.** Kullanıcı Notion'ı hem
   Manus'a hem Claude Code'a (bu oturumdaki `mcp__9818bd72-...__notion-*`
   araçları) bağladığında, Manus'un yazdığı sayfa `notion-search` ile
   dakikalar içinde bulunabildi ve `notion-fetch` ile tam içerik güvenilir
   şekilde okunabildi — hiç oturum/senkron sorunu çıkmadı.

## Kurallar

1. **Manus prompt'unun İÇİNE, en başından, açıkça şunu yaz:**
   > "ÖNEMLİ: Sonucu SADECE sohbette değil, Notion'a bağlı entegrasyonun
   > üzerinden bir Notion sayfasına da yaz (tablo formatında). Notion'a
   > yazdıktan sonra o sayfanın linkini de sohbette paylaş."

   Bunu unutup göreve başladıktan sonra "aslında Notion'a da yaz" demek
   işe yaramaz — o an zaten aktif olan görev talimatını değiştiremez, yeni
   bir görev açman gerekir. Baştan ekle.

2. **Mode seçimi:** Basit/az sayıda sonuç istenen görevler için
   `mode: "speed"` (dakikalar içinde biter). Derin/çok kaynaklı doğrulama
   gereken görevler için `mode: "quality"` (5-10+ dakika sürebilir, sabırlı
   ol). Kapsamı büyütmek yerine (`örn. 20 şirket + tam doğrulama`) görevi
   küçük tut — büyük kapsam + quality mode kombinasyonu görevi gerçekten
   uzun süre "takılı" bıraktı (canlı olay: 20 şirket isteği ~15 dakikadan
   uzun sürdü ve kullanıcı sabrı tükendi; 8-10 şirket + speed mode aynı işi
   dakikalar içinde bitirdi).

3. **Bekleme döngüsü — Browser pane DEĞİL, Notion sorgula:**
   - `mcp__manus-mcp__create_task` çağrısından sonra `ScheduleWakeup` ile
     120-180 saniye sonrasına bir uyanma kur (görev karmaşıklığına göre
     150-300s arası).
   - Uyanınca ÖNCE `notion-search` (görev konusuyla ilgili 1-2 anahtar
     kelime) veya `notion-list-recent-pages` çağır. Manus'un yazdığı
     sayfa genelde görev başlığına yakın bir isimle görünür.
   - Bulunursa `notion-fetch` ile tam içeriği oku, kullanıcıya sun.
   - Bulunamazsa ve görev muhtemelen bitmemişse, tekrar `ScheduleWakeup`
     kur (aynı süre kadar veya biraz uzun). Browser pane'e SADECE Notion
     araması art arda 2-3 kez boş dönerse ve kullanıcı özellikle isterse
     son çare olarak başvur — o zaman da önce `tabs_context` ile oturumun
     hâlâ açık olduğunu doğrula (`"Giriş"` başlığı görürsen oturum düşmüş
     demektir, kullanıcıdan tekrar giriş istemekten başka çare yok).

4. **`notion-search` bazen filtre hatası verir** ("Some requested search
   filters aren't available for this connection"). Bu durumda `sort` gibi
   parametreleri bırak, sade `query` ile tekrar dene.

5. **Farklı Notion hesabı ihtimali.** Eğer Notion araması hep boş dönüyorsa
   ve kullanıcı "orada görünüyor" diyorsa, Claude Code'un bağlı olduğu
   Notion hesabı ile Manus'un yazdığı hesap **farklı** olabilir (canlı
   olay: ilk denemede Claude'un gördüğü workspace boş bir demo alanıydı —
   "Welcome to Notion!", "Student Planner" gibi örnek sayfalar — kullanıcı
   sonra Notion'ı Claude Code'a da açıkça bağladı ve o andan sonra
   çalıştı). Kullanıcıya bunu netleştir, gerekirse Notion'ı bu oturuma da
   bağlamasını iste.

## Akış Özeti

```
1. Görev tanımını netleştir (ne aranacak, kaç sonuç, hangi kısıtlar).
2. mcp__manus-mcp__create_task çağır:
   - prompt: iş tanımı + "ÖNEMLİ: ... Notion'a da yaz ..." talimatı
   - mode: speed (basit) veya quality (derin doğrulama)
3. ScheduleWakeup (150-300s, göreve göre).
4. Uyanınca: notion-search(query) -> bulunursa notion-fetch(id) -> sun.
   Bulunamazsa: ScheduleWakeup ile tekrar dene (Browser pane'e dönme).
5. Sonucu kullanıcıya tablo/özet olarak sun, kaynak sayfanın Notion linkini
   paylaş.
```
