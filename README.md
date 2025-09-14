🚀 Derleme ve Kullanım
1. Derleme
bash

go mod init elm37-reader
go mod tidy
go build -o elm37-reader

2. Çalıştırma
bash

# Basit kullanım
./elm37-reader 192.168.0.10 35000

# Özel timeout ile
./elm37-reader 192.168.0.10 35000 10

# TLS ile (eğer destekleniyorsa)
./elm37-reader --tls 192.168.0.10 35000

3. Docker ile Çalıştırma
dockerfile

FROM golang:1.21-alpine

WORKDIR /app
COPY . .
RUN go build -o elm37-reader

CMD ["./elm37-reader", "192.168.0.10", "35000"]

📋 Özellikler

    ✅ ELM327/ELM37 TCP bağlantısı

    ✅ AT komutları desteği

    ✅ OBD-II PID okuma

    ✅ Veri dönüştürme ve parsing

    ✅ Timeout yönetimi

    ✅ TLS/SSL desteği

    ✅ Detaylı logging

    ✅ Hata yönetimi

🔧 Yapılandırma

Çevre değişkenleri ile yapılandırma:
bash

export ELM37_HOST="192.168.0.10"
export ELM37_PORT="35000"
export ELM37_TIMEOUT="5"
export ELM37_USE_TLS="false"

Bu kod, ELM37 cihazına TCP üzerinden bağlanır, OBD-II verilerini okur ve insan tarafından okunabilir formata dönüştürür.
