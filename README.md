# ELM37 OBD-II Reader 🚗💻

ELM37 cihazına bağlanarak araç verilerini okuyan Go dilinde yazılmış gelişmiş bir OBD-II okuyucu uygulaması.

## 📦 Kurulum

### Kaynaktan Derleme

```bash
# Modülü başlat
go mod init elm37-reader
go mod tidy

# Binary dosyasını derle
go build -o elm37-reader
```

### Docker ile Çalıştırma

```dockerfile
FROM golang:1.21-alpine
WORKDIR /app
COPY . .
RUN go build -o elm37-reader
CMD ["./elm37-reader", "192.168.0.10", "35000"]
```

## 🚀 Kullanım

### Temel Kullanım

```bash
# Basit kullanım (varsayılan timeout: 5s)
./elm37-reader 192.168.0.10 35000

# Özel timeout ile (10 saniye)
./elm37-reader 192.168.0.10 35000 10

# TLS/SSL bağlantısı ile (cihaz destekliyorsa)
./elm37-reader --tls 192.168.0.10 35000
```

### Çevre Değişkenleri ile Yapılandırma

```bash
# Çevre değişkenleri ile yapılandırma
export ELM37_HOST="192.168.0.10"
export ELM37_PORT="35000"
export ELM37_TIMEOUT="5"
export ELM37_USE_TLS="false"

# Çalıştır
./elm37-reader
```

## ✨ Özellikler

- ✅ **ELM327/ELM37 TCP Bağlantısı** - Ethernet/WiFi bağlantılı cihazlarla uyumlu
- ✅ **AT Komut Desteği** - Tüm standart ELM327 AT komutlarını destekler
- ✅ **OBD-II PID Okuma** - Gerçek zamanlı araç verilerini okur
- ✅ **Akıllı Veri Dönüştürme** - Ham hex verilerini okunabilir formata çevirir
- ✅ **Timeout Yönetimi** - Bağlantı sorunlarında otomatik yeniden deneme
- ✅ **TLS/SSL Desteği** - Güvenli bağlantı seçeneği
- ✅ **Detaylı Loglama** - Tüm işlemleri ayrıntılı şekilde kaydeder
- ✅ **Hata Yönetimi** - Bağlantı kopmalarında graceful recovery

## 🔧 Desteklenen PID'ler

| PID  | Açıklama                          | Birim        |
|------|-----------------------------------|-------------|
| 0D   | Araç Hızı                         | km/h        |
| 0C   | Motor Devri                       | RPM         |
| 05   | Soğutucu Sıcaklığı                | °C          |
| 0A   | Yakıt Basıncı                     | kPa         |
| 0B   | Emme Manifoldu Basıncı            | kPa         |
| 2F   | Yakıt Seviyesi                    | %           |
| 46   | Ortam Hava Sıcaklığı              | °C          |

## 📊 Örnek Çıktı

```
[INFO] ELM37 OBD-II Reader Başlatılıyor...
[INFO] ELM37 cihazına bağlandı: 192.168.0.10:35000
[INFO] AT Yanıt (ATZ): ELM327 v2.1
[INFO] AT Yanıt (ATE0): OK

=== OBD-II Veri Okuma ===
Çıkmak için Ctrl+C basın
========================

--- Okuma 1 ---
Araç Hızı                  : 45 km/h
Motor Devri                : 1250 RPM
Soğutucu Sıcaklığı         : 87 °C
Yakıt Basıncı              : 356 kPa
Emme Manifoldu Basıncı     : 32 kPa

--- Okuma 2 ---
Araç Hızı                  : 52 km/h
Motor Devri                : 1380 RPM
Soğutucu Sıcaklığı         : 88 °C
Yakıt Basıncı              : 362 kPa
Emme Manifoldu Basıncı     : 35 kPa
```

## 🛠️ Geliştirme

### Yeni PID Eklemek

`ConvertPIDValue` fonksiyonuna yeni bir case ekleyerek desteklenen PID'leri genişletebilirsiniz:

```go
case "2F": // Fuel Level Input
    if len(values) >= 1 {
        level, _ := strconv.ParseInt(values[0], 16, 64)
        percentage := (level * 100) / 255
        return fmt.Sprintf("%d %%", percentage), nil
    }
```

### Özel AT Komutları

Özel AT komutları eklemek için `Initialize` fonksiyonunu genişletebilirsiniz:

```go
commands := []string{
    "ATZ",      // Reset
    "ATE0",     // Echo off
    "ATL0",     // Line feeds off
    "ATH1",     // Headers on
    "ATSP0",    // Auto protocol
    "AT@1",     // Özel komut (cihaza özel)
    // ... diğer komutlar
}
```

## ⚠️ Sorun Giderme

### Bağlantı Sorunları

1. **"Bağlantı hatası"** - IP ve port bilgilerini kontrol edin
2. **"Zaman aşımı"** - Timeout süresini artırın veya ağ bağlantısını kontrol edin
3. **"Komut yanıtı alınamadı"** - ELM37 cihazının doğru modda olduğundan emin olun

### Veri Okuma Sorunları

1. **"PID yanıtı bulunamadı"** - Aracınızın ilgili PID'i desteklediğinden emin olun
2. **"Geçersiz veri formatı"** - ELM37 cihaz firmware'ini güncellemeyi deneyin

## 📜 Lisans

Bu proje MIT lisansı altında lisanslanmıştır. Detaylar için LICENSE dosyasına bakın.
---

**Not:** Bu yazılım sadece diagnoz ve eğitim amaçlıdır. Araç güvenliğini etkileyen herhangi bir değişiklik yapmadan önce daima üretici talimatlarına uyunuz.
