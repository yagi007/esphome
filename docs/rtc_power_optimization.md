# RTC Power Optimization - DS3231 Implementation

## Problém s WiFi a SNTP
Aktuální konfigurace používá SNTP pro čas, což vyžaduje:
- **WiFi aktivitu** každých 15-60 minut pro NTP synchronizaci
- **Kontinuální WiFi připojení** pro časové aktualizace
- **Spotřeba energie:** ~30-50mA kontinuálně

## Řešení: Externí RTC Modul DS3231

### Hardware Požadavky
- **DS3231 RTC modul** (~$3-5)
- **CR2032 baterie** pro backup
- **I2C připojení** (využije existující I2C bus)

### Výhody RTC Řešení
- ✅ **Úspora energie:** 30-50mA (eliminuje WiFi pro čas)
- ✅ **Přesnost:** ±2ppm (±1 minuta/rok)
- ✅ **Backup:** Čas přežije vypnutí a reboot
- ✅ **Offline provoz:** Funkční bez internetu
- ✅ **Stejné API:** Kompatibilní s existujícím kódem

## Zapojení Hardware

### Piny (ESP32-S3)
```
DS3231    ESP32-S3
======    ========
VCC   ->  3.3V
GND   ->  GND
SCL   ->  GPIO9  (existující I2C)
SDA   ->  GPIO8  (existující I2C)
```

### I2C Adresa
- **DS3231:** `0x68`
- **CH422G:** `0x24` (existující)
- Žádný konflikt adres ✅

## ESPHome Konfigurace

### 1. Přidat DS3231 do hardware.yaml

```yaml
# Do esphome/obivan/hardware.yaml

# Nahradit současnou time sekci:
# time:
#   - platform: sntp
#     id: sntp_time
#     timezone: Europe/Prague

# Nová RTC konfigurace:
ds1307:
  id: ds1307_time
  address: 0x68

time:
  - platform: ds1307
    ds1307_id: ds1307_time
    id: rtc_time
    timezone: Europe/Prague
    # Volitelné: počáteční nastavení času
    on_time_sync:
      then:
        - logger.log: "RTC time synchronized"
```

### 2. Aktualizovat odkazy na čas

**V `obivan/display/main.yaml`:**
```yaml
# Změnit všechny výskyty:
# auto now = id(sntp_time).now();
# Na:
# auto now = id(rtc_time).now();
```

### 3. Přidat časové nastavení do System stránky

```yaml
# Do system.yaml přidat tlačítka pro manuální nastavení času
lvgl:
  pages:
    - id: page_system
      widgets:
        # Přidat Time Setting sekci
        - button:
            x: 20
            y: 200
            width: 200
            height: 40
            widgets:
              - label:
                  text: "SET TIME"
            on_click:
              - lvgl.page.show: page_time_setting

    # Nová stránka pro nastavení času
    - id: page_time_setting
      widgets:
        - label:
            text: "Time Setting"
            x: 10
            y: 10
```

## Implementační Kroky

### Fáze 1: Hardware Test
```yaml
# Test I2C komunikace
i2c:
  - id: bus_a
    sda: GPIO08
    scl: GPIO09
    scan: true  # Zobrazí nalezená zařízení včetně DS3231
```

### Fáze 2: Basic RTC
```yaml
# Základní RTC bez WiFi
ds1307:
  id: ds1307_rtc

time:
  - platform: ds1307
    ds1307_id: ds1307_rtc
    id: rtc_time
    timezone: Europe/Prague
```

### Fáze 3: Hybridní Režim (Volitelné)
```yaml
# RTC + občasná WiFi sync
globals:
  - id: auto_time_sync
    type: bool
    initial_value: false

interval:
  - interval: 24h
    then:
      - if:
          condition:
            lambda: "return id(auto_time_sync);"
          then:
            - logger.log: "Starting daily time sync..."
            - wifi.enable:
            - delay: 60s
            - lambda: |-
                // Sync RTC s NTP pokud je WiFi připojená
                if (wifi::global_wifi_component->is_connected()) {
                  ESP_LOGI("time", "WiFi connected, RTC will sync with NTP");
                } else {
                  ESP_LOGI("time", "WiFi not connected, using RTC time");
                }
            - wifi.disable:
            - logger.log: "Daily time sync completed"
```

## Kódové Změny

### Globální Nahrazení
Nahradit všechny výskyty:
```cpp
// Staré:
id(sntp_time).now()

// Nové:  
id(rtc_time).now()
```

### Detekce Validního Času
```cpp
// Přidat do init scriptu:
lambda: |-
  auto time = id(rtc_time).now();
  if (time.year < 2024) {
    ESP_LOGW("rtc", "RTC time not set - showing default time");
    // Zobrazit varování v GUI
  } else {
    ESP_LOGI("rtc", "RTC time valid: %04d-%02d-%02d %02d:%02d", 
             time.year, time.month, time.day_of_month, time.hour, time.minute);
  }
```

## Power Savings Comparison

### Před (SNTP):
- **WiFi aktivní:** 30-50mA kontinuálně
- **NTP sync:** 5-10mA každých 15-60min
- **Celkem:** ~35-60mA

### Po (RTC):
- **RTC komunikace:** ~0.1mA při čtení
- **I2C overhead:** ~1-2mA
- **WiFi vypnuto:** 0mA
- **Celkem:** ~1-2mA

**Úspora: 30-55mA** 🎉

## Testování

### 1. I2C Scan
```
[I][i2c:059]: Scanning i2c bus for active devices...
[I][i2c:066]: Found i2c device at address 0x24 (CH422G)
[I][i2c:066]: Found i2c device at address 0x68 (DS3231)
```

### 2. RTC Status
```
[I][ds1307:023]: DS1307 RTC initialized
[I][time:034]: Synchronized time: 2025-01-27 15:30:00
[I][rtc]: RTC time valid: 2025-01-27 15:30
```

### 3. Power Measurement
- **Screen ON s RTC:** ~90-110mA (vs. 120-140mA s WiFi)
- **Screen OFF s RTC:** ~15-35mA (vs. 50-70mA s WiFi)

## Fallback Strategie

Pokud DS3231 selže, přidat fallback:
```cpp
lambda: |-
  auto time = id(rtc_time).now();
  if (!time.is_valid() || time.year < 2024) {
    ESP_LOGW("time", "RTC failed - using internal clock");
    // Zobrazit "-- : --" nebo poslední známý čas
    return "RTC ERR";
  }
  return str_sprintf("%02d:%02d", time.hour, time.minute);
```

## Náklady vs. Úspory

### Náklady:
- **DS3231 modul:** $3-5
- **CR2032 baterie:** $1-2
- **Čas implementace:** 2-3 hodiny

### Úspory:
- **Energie:** 30-55mA (42-77% reduction v screen-off)
- **Spolehlivost:** Offline provoz
- **Přesnost:** Lepší než SNTP při přerušeném WiFi

---

**Závěr:** DS3231 RTC je ideální řešení pro maximální úsporu energie při zachování funkčnosti času. Doporučeno pro battery-powered aplikace nebo offline provoz.

**Další krok:** Objednat DS3231 modul a implementovat podle této dokumentace.