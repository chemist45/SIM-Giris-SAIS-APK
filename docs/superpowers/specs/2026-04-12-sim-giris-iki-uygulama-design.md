# SİM Giriş — İki Ayrı Uygulama Tasarımı

**Tarih:** 2026-04-12  
**Proje:** İZSU Yarımada Tesisler SİM Giriş Sistemi  
**Kapsam:** SAİS Sorumlusu ve Alıcılar için ayrı, sadeleştirilmiş uygulamalar

---

## Genel Bakış

Mevcut 6 adet tesis HTML dosyası (cesme-aat.html vb.) her şeyi içeriyor: alıcı seçimi, SMS kurulumu, admin paneli. Bu tasarım, kullanıcı rolüne göre iki ayrı, sade uygulama üretir.

---

## Yeni Dosyalar

### 1. `sais-app.html` — SAİS Sorumlusu Uygulaması

**Kim kullanır:** 2FA SMS'in geldiği telefonu olan kişi (her tesis için 1 kişi)

**Ekranlar:**

**Ekran 1 — Tesis Seçimi**
- 6 tesis kartı (2 sütun grid)
- Her kart: tesis adı + renk aksanı

**Ekran 2 — Tesis Detayı (tesis seçilince)**
- `✅ SMS İletimi Aktif` banner (Android APK modunda)
- `🔗 Bağlantı Testi` butonu → GAS'a `ping` gönderir, sonucu gösterir
- **iPhone bölümü** (sadece iOS kullanıcısına gösterilir):
  - `📥 iOS Kısayolunu İndir` butonu (tek tıkla .shortcut dosyası indirir)
  - 3 adımlı kısa rehber:
    1. İndirilen dosyayı aç → Kısayollar uygulamasına ekle
    2. Otomasyon → Mesaj (içerir: sim.csb.gov.tr) → "SİM SMS İlet" çalıştır
    3. "Çalıştırmadan Önce Sor" → Kapat

**Notlar:**
- ntfy, Telegram, WhatsApp gösterilmez
- MacroDroid adımları gösterilmez (APK otomatik halleder)
- Admin paneli yok

---

### 2. `alici-app.html` — Alıcı Uygulaması

**Kim kullanır:** 2FA kodunu görmek isteyen tüm alıcılar (Android, iOS, masaüstü)

**Ekranlar:**

**Ekran 1 — Tesis Seçimi**
- 6 tesis kartı (2 sütun grid)
- Tesis adı + renk aksanı

**Ekran 2 — Giriş ve Kod (tesis seçilince)**

*Giriş Paneli:*
- `KULLANICI ADI` → tıklayınca kopyalanır
- `ŞİFRE` → otomatik panoya kopyalanır, ekranda görünür
- `🔐 SİM GİRİŞ YAP` butonu:
  - `setRecipient` GAS'a gönderilmez (alıcı seçimi yok)
  - `sim.csb.gov.tr` yeni sekmede açılır
  - Kod paneli otomatik açılır

*Kod Paneli (`📬 2FA Kodunu Getir`):*
- `Kodu Getir 🔄` butonu → GAS'a `getCode` POST gönderir
- **Her 15 saniyede otomatik yeniler** (panel açıkken)
- Kod gösterimi:
  - Büyük, belirgin font
  - Kaç saniye önce geldiği
  - Renk: 🟢 0-2 dk, 🟡 2-4 dk, 🔴 4-5 dk
  - 5 dk sonrası: "Kod süresi doldu"
- Kod yok durumu: "⏳ Henüz kod gelmedi"

**Notlar:**
- Ad seçimi yok — kod tesis bazlı saklanır, herkes görebilir
- ntfy, Telegram, WhatsApp gösterilmez
- SMS kurulum adımları yok
- Masaüstü tarayıcıda tam çalışır

---

## Mevcut Dosyalar

| Dosya | Durum | Açıklama |
|-------|-------|----------|
| `cesme-aat.html` vb. (6 adet) | Korunur | Admin paneli, alıcı/şifre yönetimi |
| `gas-kod.js` | Güncellenir | `getCode` eklenir, `smsWebhook` kodu kaydeder |
| APK `index.html` | Güncellenir | "SAİS Sorumlusu" ve "Alıcı Girişi" seçenekleri |

---

## GAS Değişiklikleri

### Yeni: `getCode` aksiyonu

**İstek:**
```json
{ "action": "getCode", "tesis": "cesme" }
```

**Başarılı yanıt:**
```json
{ "ok": true, "body": "Giriş kodunuz: 123456", "ageSec": 47 }
```

**Kod yok:**
```json
{ "ok": false, "message": "Henüz kod yok" }
```

**Süresi dolmuş (>300 sn):**
```json
{ "ok": false, "expired": true, "ageSec": 340 }
```

### Değişen: `smsWebhook`

Mevcut bildirim gönderme işlemlerine ek olarak, gelen SMS içeriğini Script Properties'e kaydeder:

```
Anahtar: lastCode_[tesisId]
Değer:   { "body": "...", "timestamp": "ISO string" }
```

Örnek anahtarlar: `lastCode_cesme`, `lastCode_urla`, `lastCode_mordogan`

### Değişmeyen

- `setRecipient`, `ping`, `adminLogin`, `updateRecipients`, `updateCredentials`
- ntfy, Telegram, WhatsApp, SMS gönderme fonksiyonları (GAS'ta kalır, yeni uygulamalarda gösterilmez)

---

## Platform Desteği

| Platform | sais-app.html | alici-app.html |
|----------|--------------|----------------|
| Android APK | ✅ SMS otomatik | ✅ Tam çalışır |
| Android tarayıcı | ✅ (SMS aktif değil) | ✅ Tam çalışır |
| iPhone Safari | ✅ + iOS Shortcuts rehberi | ✅ Tam çalışır |
| Masaüstü tarayıcı | ✅ (SMS aktif değil) | ✅ Tam çalışır |

---

## iOS SMS Kurulumu (SAİS Sorumlusu)

Apple kısıtlaması nedeniyle SMS otomasyonu otomatik kurulamaz. Minimum kurulum:

1. `sais-app.html`'de **"📥 iOS Kısayolunu İndir"** → dosyayı aç → Ekle
2. Kısayollar → Otomasyon → + → Kişisel Otomasyon → **Mesaj** → içerik: `sim.csb.gov.tr` → **Kısayolu Çalıştır** → "SİM SMS İlet [Tesis]"
3. "Çalıştırmadan Önce Sor" → **Kapat**

Tek seferlik kurulum. Sonrası otomatik.

---

## Uygulama Planı (Sıra)

1. `gas-kod.js` güncelle — `getCode` ekle, `smsWebhook`'a kod kaydetme ekle
2. `alici-app.html` oluştur
3. `sais-app.html` oluştur
4. APK `index.html` güncelle — iki mod ekle
5. APK assets'e yeni HTML'leri ekle
6. GitHub'a push → yeni APK build
