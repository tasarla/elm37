package main

import (
    "bufio"
    "context"
    "crypto/tls"
    "fmt"
    "log"
    "net"
    "os"
    "strconv"
    "strings"
    "time"
)

// ELM37 cihazından veri okumak için yapılandırma
type ELM37Config struct {
    Host        string
    Port        int
    Timeout     time.Duration
    UseTLS      bool
    TLSConfig   *tls.Config
    AutoConnect bool
    Commands    []string // Gönderilecek AT komutları
}

// ELM37 istemci yapısı
type ELM37Client struct {
    conn    net.Conn
    config  *ELM37Config
    scanner *bufio.Scanner
    writer  *bufio.Writer
}

// Yeni ELM37 istemcisi oluştur
func NewELM37Client(config *ELM37Config) *ELM37Client {
    return &ELM37Client{
        config: config,
    }
}

// ELM37 cihazına bağlan
func (c *ELM37Client) Connect() error {
    address := fmt.Sprintf("%s:%d", c.config.Host, c.config.Port)
    
    var conn net.Conn
    var err error
    
    ctx, cancel := context.WithTimeout(context.Background(), c.config.Timeout)
    defer cancel()
    
    if c.config.UseTLS {
        dialer := &tls.Dialer{
            Config: c.config.TLSConfig,
        }
        conn, err = dialer.DialContext(ctx, "tcp", address)
    } else {
        dialer := &net.Dialer{}
        conn, err = dialer.DialContext(ctx, "tcp", address)
    }
    
    if err != nil {
        return fmt.Errorf("bağlantı hatası: %w", err)
    }
    
    c.conn = conn
    c.scanner = bufio.NewScanner(conn)
    c.writer = bufio.NewWriter(conn)
    
    log.Printf("ELM37 cihazına bağlandı: %s", address)
    return nil
}

// Komut gönder ve yanıt al
func (c *ELM37Client) SendCommand(command string) (string, error) {
    if c.conn == nil {
        return "", fmt.Errorf("bağlantı yok")
    }
    
    // Komutu gönder
    fullCommand := fmt.Sprintf("%s\r\n", command)
    _, err := c.writer.WriteString(fullCommand)
    if err != nil {
        return "", fmt.Errorf("komut gönderme hatası: %w", err)
    }
    
    err = c.writer.Flush()
    if err != nil {
        return "", fmt.Errorf("flush hatası: %w", err)
    }
    
    log.Printf("Komut gönderildi: %s", strings.TrimSpace(fullCommand))
    
    // Yanıtı oku
    var response strings.Builder
    timeout := time.After(c.config.Timeout)
    
    // ELM37 cihazları genellikle ">" prompt'u veya son satırda yanıt verir
    for {
        select {
        case <-timeout:
            return response.String(), fmt.Errorf("zaman aşımı")
        default:
            if c.scanner.Scan() {
                line := c.scanner.Text()
                response.WriteString(line + "\n")
                
                // ELM37 prompt'u geldi mi kontrol et
                if strings.Contains(line, ">") || strings.Contains(line, "OK") || strings.Contains(line, "ERROR") {
                    return response.String(), nil
                }
                
                // Bazı komutlar belirli terminatörlerle biter
                if strings.Contains(line, "SEARCHING...") || strings.Contains(line, "STOPPED") {
                    return response.String(), nil
                }
            } else {
                if err := c.scanner.Err(); err != nil {
                    return response.String(), fmt.Errorf("okuma hatası: %w", err)
                }
                return response.String(), nil
            }
        }
    }
}

// OBD-II verilerini oku
func (c *ELM37Client) ReadOBDData(pid string) (string, error) {
    // PID formatını kontrol et
    if len(pid) != 2 {
        return "", fmt.Errorf("geçersiz PID: %s", pid)
    }
    
    command := fmt.Sprintf("01 %s", pid)
    response, err := c.SendCommand(command)
    if err != nil {
        return "", err
    }
    
    return c.ParseOBDResponse(response, pid)
}

// OBD yanıtını parse et
func (c *ELM37Client) ParseOBDResponse(response, pid string) (string, error) {
    lines := strings.Split(response, "\n")
    
    for _, line := range lines {
        line = strings.TrimSpace(line)
        
        // Yanıt satırlarını ara (41 [PID] formatında)
        if strings.HasPrefix(line, "41 "+pid) {
            // Hex değerleri al
            parts := strings.Fields(line)
            if len(parts) >= 3 {
                return c.ConvertPIDValue(pid, parts[2:])
            }
        }
        
        // Bazı cihazlar farklı formatlarda yanıt verebilir
        if strings.Contains(line, pid) && len(line) >= 10 {
            // Örnek: "41 0C 1F 88" -> Engine RPM
            return line, nil
        }
    }
    
    return "", fmt.Errorf("PID yanıtı bulunamadı: %s", pid)
}

// PID değerlerini dönüştür
func (c *ELM37Client) ConvertPIDValue(pid string, values []string) (string, error) {
    switch pid {
    case "0D": // Vehicle Speed
        if len(values) >= 1 {
            speed, err := strconv.ParseInt(values[0], 16, 64)
            if err != nil {
                return "", err
            }
            return fmt.Sprintf("%d km/h", speed), nil
        }
        
    case "0C": // Engine RPM
        if len(values) >= 2 {
            a, _ := strconv.ParseInt(values[0], 16, 64)
            b, _ := strconv.ParseInt(values[1], 16, 64)
            rpm := float64((a*256)+b) / 4.0
            return fmt.Sprintf("%.0f RPM", rpm), nil
        }
        
    case "05": // Engine Coolant Temperature
        if len(values) >= 1 {
            temp, _ := strconv.ParseInt(values[0], 16, 64)
            celsius := temp - 40
            return fmt.Sprintf("%d °C", celsius), nil
        }
        
    default:
        // Ham hex değerleri döndür
        return strings.Join(values, " "), nil
    }
    
    return "", fmt.Errorf("geçersiz veri formatı")
}

// Bağlantıyı kapat
func (c *ELM37Client) Close() error {
    if c.conn != nil {
        return c.conn.Close()
    }
    return nil
}

// ELM37 cihazını başlat ve test et
func (c *ELM37Client) Initialize() error {
    // AT komutları ile cihazı hazırla
    commands := []string{
        "ATZ",      // Reset
        "ATE0",     // Echo off
        "ATL0",     // Line feeds off
        "ATH1",     // Headers on
        "ATSP0",    // Auto protocol
    }
    
    for _, cmd := range commands {
        response, err := c.SendCommand(cmd)
        if err != nil {
            return fmt.Errorf("AT komutu hatası (%s): %w", cmd, err)
        }
        log.Printf("AT Yanıt (%s): %s", cmd, strings.TrimSpace(response))
        time.Sleep(100 * time.Millisecond)
    }
    
    return nil
}

// Örnek kullanım
func main() {
    log.Println("ELM37 OBD-II Reader Başlatılıyor...")
    
    // ELM37 yapılandırması
    config := &ELM37Config{
        Host:    "192.168.0.10",  // ELM37 cihaz IP adresi
        Port:    35000,           // Varsayılan port
        Timeout: 5 * time.Second,
        UseTLS:  false,
        Commands: []string{
            "ATZ", "ATE0", "ATL0", "ATH1", "ATSP0",
        },
    }
    
    // İstemci oluştur
    client := NewELM37Client(config)
    
    // Bağlan
    err := client.Connect()
    if err != nil {
        log.Fatalf("Bağlantı hatası: %v", err)
    }
    defer client.Close()
    
    // Cihazı başlat
    err = client.Initialize()
    if err != nil {
        log.Fatalf("Başlatma hatası: %v", err)
    }
    
    // Okuyacağımız PID'ler
    pids := map[string]string{
        "0D": "Vehicle Speed",
        "0C": "Engine RPM", 
        "05": "Engine Coolant Temperature",
        "0A": "Fuel Pressure",
        "0B": "Intake Manifold Pressure",
    }
    
    fmt.Println("\n=== OBD-II Veri Okuma ===")
    fmt.Println("Çıkmak için Ctrl+C basın")
    fmt.Println("========================")
    
    // Sürekli veri oku
    for i := 0; i < 10; i++ { // 10 kez oku, sonsuz döngü için for {...} kullan
        fmt.Printf("\n--- Okuma %d ---\n", i+1)
        
        for pid, description := range pids {
            value, err := client.ReadOBDData(pid)
            if err != nil {
                fmt.Printf("%-30s: Hata - %v\n", description, err)
            } else {
                fmt.Printf("%-30s: %s\n", description, value)
            }
            time.Sleep(500 * time.Millisecond)
        }
        
        time.Sleep(2 * time.Second)
    }
    
    log.Println("Program sonlandırıldı")
}

// TLS yapılandırması (isteğe bağlı)
func createTLSConfig() *tls.Config {
    return &tls.Config{
        InsecureSkipVerify: true, // Sadece test için
        MinVersion:         tls.VersionTLS12,
    }
}

// Komut satırı arayüzü
func parseArgs() *ELM37Config {
    if len(os.Args) < 3 {
        fmt.Printf("Kullanım: %s <host> <port> [timeout_seconds]\n", os.Args[0])
        fmt.Println("Örnek: ./elm37-reader 192.168.0.10 35000 5")
        os.Exit(1)
    }
    
    host := os.Args[1]
    port, err := strconv.Atoi(os.Args[2])
    if err != nil {
        log.Fatalf("Geçersiz port: %v", err)
    }
    
    timeout := 5 * time.Second
    if len(os.Args) > 3 {
        sec, err := strconv.Atoi(os.Args[3])
        if err == nil {
            timeout = time.Duration(sec) * time.Second
        }
    }
    
    return &ELM37Config{
        Host:    host,
        Port:    port,
        Timeout: timeout,
        UseTLS:  false,
    }
}
