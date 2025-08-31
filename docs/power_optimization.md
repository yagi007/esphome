# Power Optimization Summary - ESPHome ESP32-S3

## Problém
Vysoká spotřeba energie v režimu screen-off (přes 100mA) místo očekávaných 50-80mA.

## Omezení Hardware
- **ESP32-S3 s PSRAM 120MHz vyžaduje CPU frekvenci 240MHz**
- Snížení CPU pod 240MHz způsobí nefunkčnost PSRAM
- Řešení: Snížení PSRAM z 120MHz na 80MHz umožní CPU scaling

## Implementované Optimalizace

### 1. PSRAM Frekvence
**Před:**
```yaml
psram:
  mode: octal
  speed: 120MHz
CONFIG_SPIRAM_SPEED_120M: y
```

**Po:**
```yaml
psram:
  mode: octal
  speed: 80MHz
CONFIG_SPIRAM_SPEED_80M: y
```

**Úspora:** ~15-20mA

### 2. Power Management v ESP-IDF
```yaml
CONFIG_PM_ENABLE: y
CONFIG_FREERTOS_USE_TICKLESS_IDLE: y
CONFIG_PM_DFS_INIT_AUTO: y
```

### 3. Software Optimalizace v Screen-Off Mode

#### A. CPU Load Reduction
```cpp
// Místo komplexního esp_pm_configure API
id(aggressive_power_saving) = true;
id(low_power_cpu_mode) = true;
```

#### B. WiFi Power Save Optimalizace
```cpp
// Screen OFF
wifi_set_ps(WIFI_PS_MAX_MODEM);  // Maximální úspora WiFi

// Screen ON  
wifi_set_ps(WIFI_PS_MIN_MODEM);  // Normální režim
```

**Úspora:** ~10-15mA

#### C. Logging Reduction
```cpp
// Screen OFF
esp_log_level_set("*", ESP_LOG_WARN);

// Screen ON
esp_log_level_set("*", ESP_LOG_DEBUG);
```

**Úspora:** ~5mA

#### D. Sensor Update Intervals
- Zpomalení aktualizací senzorů když je obrazovka vypnutá
- WiFi signal sensor má delší interval

**Úspora:** ~5mA

### 4. Už Implementované Optimalizace
```yaml
# WiFi základní optimalizace
wifi:
  power_save_mode: LIGHT
  passive_scan: True

# Compiler optimalizace
CONFIG_COMPILER_OPTIMIZATION_SIZE: y
```

## Celková Očekávaná Úspora

| Komponenta | Před | Po | Úspora |
|------------|------|----|---------| 
| PSRAM | 25mA (120MHz) | 15mA (80MHz) | 10mA |
| CPU Load | 40mA (240MHz plná zátěž) | 25mA (redukovaná zátěž) | 15mA |
| WiFi | 30mA (normální) | 20mA (max save) | 10mA |
| Logging/Processing | 10mA | 5mA | 5mA |
| **CELKEM** | **105mA** | **65mA** | **40mA** |

## Testování

### Měření Spotřeby
1. **Screen ON:** ~120-140mA (normální)
2. **Screen OFF před optimalizací:** ~100-120mA
3. **Screen OFF po optimalizaci:** **50-70mA** (cíl)

### Test Sequence
```yaml
# 1. Boot -> Screen ON
- logger.log: "MEASURE NOW: Boot complete, screen ON"

# 2. Screen OFF trigger
- script.execute: screen_off_mode
- delay: 5s  
- logger.log: "MEASURE NOW: Expected 50-70mA total consumption"

# 3. Screen ON restore
- script.execute: screen_on_mode
- logger.log: "MEASURE NOW: Back to normal ~120mA"
```

## Debugging

### Kritické Kontroly
1. **PSRAM funkčnost:** Po snížení na 80MHz zkontrolovat LVGL a velké alokace
2. **I2C stabilita:** CH422G komunikace po power changes
3. **WiFi konektivita:** Ověřit že max power save neruší spojení
4. **Touchscreen:** Wake-up funkčnost při power optimalizacích

### Log Monitoring
```
screen_off: CPU optimization: Extended task delays active
screen_off: WiFi set to maximum power save mode  
screen_off: Sensor update intervals optimized
screen_off: Logging reduced to WARNING level
```

## Fallback Řešení

Pokud 80MHz PSRAM způsobí problémy:

### Plán B: Pouze Software Optimalizace
```yaml
# Ponech PSRAM 120MHz, CPU 240MHz
# Optimalizuj jen:
- WiFi power save
- Sensor intervals  
- Task delays
- Logging levels
```

**Očekávaná úspora:** ~20-25mA (méně, ale bezpečnější)

## Hardware Poznámky

- **ESP32-S3-DevKitC-1** s 8MB Flash
- **PSRAM:** Podpora až 16MB, optimalizováno pro power efficiency
- **Display:** 7" RGB666, významný spotřebič energie (~80mA)
- **WiFi:** 2.4GHz, optimalizace power save modes kritická

## Další Možné Optimalizace

1. **Deep Sleep Mode:** Kompletnější suspend pro dlouhé neaktivity
2. **GPIO Power Down:** Unused pins optimization  
3. **Peripheral Gating:** Vypnutí nepoužívaných periferií
4. **DVFS (Dynamic Voltage/Frequency Scaling):** Pokud ESP-IDF umožní

---

**Autor:** Power Optimization Team  
**Datum:** 2025-01-27  
**Verze:** 1.0  
**ESPHome:** 2025.4.2+  
