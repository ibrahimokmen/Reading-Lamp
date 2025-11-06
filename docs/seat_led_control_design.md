# Yolcu Koltuğu LED Parlaklık Kontrol Sistemi Tasarımı

## 1. Sistem Özeti ve Gereksinimler

Yolcu koltuklarına entegre edilen bu sistem, kullanıcının butona ardışık basış sayısına göre LED okuma lambasının parlaklık seviyesini ayarlar. Tasarım, UNECE R10 kapsamında elektromanyetik uyumluluk (EMC) ve elektriksel dayanım testlerinden başarıyla geçecek şekilde hazırlanmıştır. Sistem 12 V otomotiv beslemesiyle çalışır, ISO 7637-2 ve ISO 16750-2 gibi ilgili test profillerine uygun koruma önlemleri içerir.

**Fonksiyonel Gereksinimler**
- Kullanıcı butona art arda basarak dört mod arasında geçiş yapar: Kapalı, Düşük, Orta, Yüksek parlaklık.
- Sistemin açılışında son kullanılan parlaklık seviyesi hatırlanır.
- LED parlaklığı titreşimsiz (flicker-free) olmalıdır.
- Butona uzun basışta "gece modu" olarak çok düşük parlaklığa alınır.

**Uyumluluk ve Güvenlik Gereksinimleri**
- UNECE R10 ve ISO 7637-2 testlerinde tanımlanan geçici gerilim darbelerine karşı giriş koruması.
- Ters kutuplama ve aşırı gerilim korumaları.
- LED sürücüsünde aşırı akım, kısa devre ve termal koruma.
- Elektromanyetik emisyonların sınırlandırılması için filtreleme ve PCB tasarım kuralları.
- Otomotiv EMC gereksinimleri doğrultusunda metal gövdeye uygun topraklama.

## 2. Donanım Tasarımı

### 2.1 Blok Diyagramı

```
12 V Araç Beslemesi
        │
   Giriş Koruma (sigorta, TVS diyot, LC filtre)
        │
  Düşürücü Regülatör (Buck, 5 V)
        │
  LDO (3.3 V) ──> Mikrodenetleyici (ARM Cortex-M0+)
        │                      │
        │                      ├─> Buton girişi (debounce RC + ESD koruma)
        │                      └─> PWM çıkışı
        │
  LED Sürücü (yüksek taraf akım sürücüsü)
        │
      LED Şerit / COB Modül
```

### 2.2 Ana Bileşen Seçimleri
- **Giriş Koruması:** AEC-Q101 onaylı TVS diyot (ör. SM8S36A), seri sigorta veya PTC, LC EMI filtresi.
- **Buck Dönüştürücü:** AEC-Q100 uygun, geniş giriş aralıklı senkron buck regülatörü (ör. TI LM2841-Q1). Girişte Pi filtre, çıkışta düşük ESR kondansatörler.
- **LDO:** Gürültü hassasiyetine karşı AEC-Q100 sertifikalı 3.3 V LDO (ör. TI TLV70033-Q1).
- **Mikrodenetleyici:** ARM Cortex-M0+ (ör. NXP S32K1xx veya Microchip ATSAMC21). Dahili EEPROM/Flash, düşük güç modları, otomotiv AEC-Q100.
- **LED Sürücü:** Sabit akım buck LED sürücü veya MOSFET + akım algı direnci (ör. TI TPS92512-Q1). PWM modülasyonu ile 200 Hz üzeri frekans.
- **Buton:** AEC-Q200 uyumlu mekanik buton, RC filtre ve ESD diyotu (ESD9M5V).

### 2.3 EMC ve R10 Uygulamaları
- Çok katmanlı PCB, analog/dijital toprak ayrımı, yıldız noktası bağlantısı.
- PWM çıkış yükseliş sürelerini sınırlayan gate sürücü/snubber.
- LED kablo demetinde bükümlü çift ve ekranlama.
- Besleme hatlarında common-mode şok bobinleri.
- Mikrodenetleyici kristali yerine dahili osilatör veya otomotiv sınıfı kristal + kapasitör.

### 2.4 Güç Yönetimi ve Koruma
- Besleme açılış/kapama sıralaması: mikrodenetleyici LDO üzerinden, LED sürücü enable pini MCU kontrolünde.
- NTC sensör veya mikrodenetleyici ADC üzerinden ısı izleme, 85 °C üzerinde parlaklığı azaltma.
- EEPROM yedek kopyası ile güç kesilmesinde veri bütünlüğü.

## 3. Yazılım Tasarımı

### 3.1 Yazılım Mimarisi
- **Sürücüler Katmanı:** GPIO, PWM, ADC, EEPROM, zamanlayıcı, kesme yönetimi.
- **Servisler Katmanı:** Buton durumu, parlaklık kontrol, termal yönetim, güç yönetimi.
- **Uygulama Katmanı:** Durum makinesi ile mod geçişleri ve kullanıcı arayüzü.

### 3.2 Durum Makinesi
| Durum | Açıklama | PWM Görevi | EEPROM İşlemi |
| --- | --- | --- | --- |
| `OFF` | LED kapalı | 0% | Son durum kaydı `OFF` |
| `LOW` | Düşük parlaklık | 15% | Son durum `LOW` |
| `MEDIUM` | Orta parlaklık | 45% | Son durum `MEDIUM` |
| `HIGH` | Yüksek parlaklık | 90% | Son durum `HIGH` |
| `NIGHT` | Uzun basış sonrası | 5% | Son durum `NIGHT` |

- Kısa basış (<400 ms) durumları sırasıyla `OFF → LOW → MEDIUM → HIGH → OFF` olarak döngüler.
- Uzun basış (>1.2 s) bulunduğu moddan `NIGHT` moduna geçirir, ikinci uzun basış çıkış.
- Termal veya düşük voltaj koruması tetiklenirse durum `SAFE_DIM` (30%) olarak zorlanır.

### 3.3 Zamanlama ve Görevler
- **Ana döngü:** Durum makinesi, LED parlaklık komutları, hata bayrakları.
- **1 ms SysTick kesmesi:** Buton debounce sayaçları, uzun/kısa basış zamanlayıcıları.
- **10 ms Görevleri:** ADC okuma (sıcaklık, besleme), filtreleme, hata kontrolü.
- **100 ms Görevleri:** EEPROM güncelleme (yalnızca durum değişince), diagnostik mesaj üretimi.

### 3.4 EMC ve Güvenilirlik İçin Yazılım Önlemleri
- PWM frekansı 400 Hz–1 kHz aralığında sabit tutulur, duty cycle güncellemeleri zero-cross sincronizasyonu ile.
- Brown-out kesmesi etkin, yeniden başlatmada son duty cycle sıfırlanır.
- Watchdog 200 ms periyotta beslenir; gömülü yazılım ISO 26262 ASIL-A uyumlu kodlama standartlarını takip eder.
- Hata günlükleri (sıcaklık, aşırı akım, EEPROM hatası) non-volatile belleğe kaydedilir.

### 3.5 Örnek Kod Parçası (C)
```c
void Button_Task(void) {
    static uint16_t press_time = 0;
    static bool button_prev = false;
    bool pressed = GPIO_Read(BUTTON_PIN);

    if (pressed) {
        if (!button_prev) {
            press_time = 0;
        }
        button_prev = true;
        press_time++;
    } else if (button_prev) {
        button_prev = false;
        if (press_time < SHORT_PRESS_TICKS) {
            NextBrightnessState();
        } else if (press_time > LONG_PRESS_TICKS) {
            ToggleNightMode();
        }
    }
}
```

## 4. Test Stratejisi

### 4.1 Donanım Testleri
- UNECE R10 uyumluluğu için 200 MHz'e kadar emisyon/immünite testleri.
- ISO 7637-2 pulse 1, 2a, 2b, 3a, 3b, 4, 5a/b testleri.
- Termal darbe (-40 °C / +85 °C), titreşim testleri (ISO 16750-3).
- Aşırı gerilim, ters kutup, kısa devre testleri.

### 4.2 Yazılım Testleri
- Birim test: buton mantığı, durum makinesi geçişleri.
- HIL testleri: PWM duty cycle doğrulama, arıza simulasyonları.
- EMC testleri sırasında fonksiyonel doğrulama.
- Regresyon testleri için otomatik test iskeleti (Ceedling/Unity).

## 5. Üretim ve Bakım Hususları
- JTAG/SWD portu EMC ekranlı kapak altında.
- Firmware güncellemesi OTA yerine servis portu üzerinden yapılır.
- LED modülü kolay değiştirilebilir, konektörlü.
- Parlaklık kalibrasyon sabitleri üretimde EEPROM'a yazılır.
- Seri numarası ve yazılım versiyonu diagnostik arayüzden okunabilir.

## 6. Risk Analizi ve Azaltma
| Risk | Etki | Önleyici Önlem |
| --- | --- | --- |
| EMC testlerinde başarısızlık | Homologasyon gecikmesi | Tasarım aşamasında simülasyon, EMI filtresi, kablo ekranlama |
| EEPROM yıpranması | Son durum kaydı yapılamaz | Yazma döngüsü sınırlama, wear leveling |
| Termal runaway | LED veya PCB hasarı | Termal sensör, duty cycle sınırı |
| Buton kontak sıçraması | Yanlış mod geçişi | Donanımsal RC + yazılımsal debounce |
| Güç darbesi reseti | Kullanıcı deneyimi bozulur | Brown-out detect, bulk kapasitör |

## 7. Sonuç

Tasarlanan sistem, buton basış sayısına göre LED parlaklığını güvenilir ve EMC uyumlu şekilde kontrol eder. Donanım ve yazılım bileşenleri R10/ISO otomotiv standartlarının gerektirdiği dayanıklılık, güvenlik ve izlenebilirlik şartlarını karşılayacak şekilde yapılandırılmıştır.
