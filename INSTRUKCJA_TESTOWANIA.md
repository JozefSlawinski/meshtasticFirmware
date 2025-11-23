# Instrukcja Testowania Modułu BLE GPS

## Wymagania

### Sprzęt:
- Urządzenie Meshtastic z GPS i BLE (np. T-Beam, Heltec V3)
- Telefon Android z aplikacją Meshtastic (lub własną aplikacją)

### Oprogramowanie:
- Skompilowany firmware z modułem BleGpsModule
- Aplikacja Android do odbierania danych przez BLE

## Krok 1: Kompilacja Firmware

### Dla ESP32 (T-Beam):
```bash
cd meshtasticFirmware
pio run -e tbeam
```

### Dla innych platform:
```bash
# Zobacz dostępne środowiska
pio run --list-targets

# Kompiluj dla wybranej platformy
pio run -e <nazwa_środowiska>
```

### Sprawdzenie czy moduł jest włączony:
Moduł jest automatycznie włączony jeśli:
- GPS nie jest wykluczony (`!MESHTASTIC_EXCLUDE_GPS`)
- Bluetooth nie jest wykluczony (`!MESHTASTIC_EXCLUDE_BLUETOOTH`)

## Krok 2: Flashowanie Firmware

```bash
# Dla T-Beam
pio run -e tbeam -t upload

# Dla innych platform
pio run -e <nazwa_środowiska> -t upload
```

## Krok 3: Weryfikacja Logów

### Połączenie Serial:
```bash
# Dla T-Beam (domyślnie 115200 baud)
pio device monitor -e tbeam

# Lub użyj dowolnego terminala serial
# Port COM (Windows) lub /dev/ttyUSB* (Linux)
# Baud rate: 115200
```

### Oczekiwane logi przy starcie:
```
BleGpsModule initialized - will send position to phone every 5000 ms
```

### Oczekiwane logi podczas działania:
```
BleGpsModule: Using position from GPS object
BleGpsModule: Sent position to phone - lat=..., lon=..., time=...
```

### Logi gdy GPS nie ma lock:
```
BleGpsModule: No GPS lock, skipping position send
```

## Krok 4: Test Podstawowy

### Scenariusz 1: Wysyłanie pozycji z GPS

1. **Przygotowanie:**
   - Wgraj firmware na urządzenie
   - Umieść urządzenie na zewnątrz (dla lepszego GPS lock)
   - Poczekaj aż GPS uzyska lock (zwykle 30-60 sekund)

2. **Połączenie BLE:**
   - Włącz Bluetooth na telefonie
   - Otwórz aplikację Meshtastic (lub własną aplikację)
   - Połącz z urządzeniem Meshtastic przez BLE

3. **Weryfikacja:**
   - Sprawdź logi - powinny pojawić się komunikaty o wysyłaniu pozycji co 5 sekund
   - W aplikacji Android sprawdź czy odbierane są dane lokalizacji
   - Sprawdź czy pozycja jest aktualizowana regularnie

### Scenariusz 2: Brak GPS Lock

1. **Przygotowanie:**
   - Umieść urządzenie w pomieszczeniu bez dostępu do GPS
   - Lub zasłoń antenę GPS

2. **Weryfikacja:**
   - Sprawdź logi - powinny pojawić się komunikaty:
     - `BleGpsModule: No GPS lock, skipping position send`
   - Moduł nie powinien wysyłać pakietów
   - Po uzyskaniu GPS lock, moduł powinien zacząć wysyłać

### Scenariusz 3: Fixed Position

1. **Konfiguracja:**
   - Skonfiguruj fixed position w Meshtastic (przez aplikację lub AdminMessage)
   - Ustaw dowolną pozycję (np. 52.2297° N, 21.0122° E dla Warszawy)

2. **Weryfikacja:**
   - Moduł powinien wysyłać fixed position nawet bez GPS lock
   - Sprawdź logi - powinny pokazywać wysyłanie pozycji

### Scenariusz 4: Brak Połączenia BLE

1. **Przygotowanie:**
   - Nie łącz aplikacji Android z urządzeniem
   - Lub rozłącz połączenie BLE

2. **Weryfikacja:**
   - Moduł powinien działać normalnie
   - Pakiety będą buforowane w kolejce `toPhoneQueue`
   - Po połączeniu aplikacji, pakiety powinny być wysłane

## Krok 5: Testy Wydajności

### Test 1: Częstotliwość wysyłania
- Sprawdź czy moduł wysyła dokładnie co 5 sekund
- Użyj logów z timestampami do weryfikacji

### Test 2: Zużycie pamięci
- Monitoruj dostępną pamięć podczas działania modułu
- Sprawdź czy nie ma wycieków pamięci

### Test 3: Zużycie baterii
- Zmierz zużycie baterii z włączonym modułem
- Porównaj z działaniem bez modułu

## Krok 6: Debugowanie

### Problem: Moduł nie wysyła pozycji

**Sprawdź:**
1. Czy moduł się zainicjalizował:
   ```
   BleGpsModule initialized - will send position to phone every 5000 ms
   ```

2. Czy GPS ma lock:
   - Sprawdź logi GPS
   - Sprawdź `gpsStatus->getHasLock()`

3. Czy pozycja jest ważna:
   - Sprawdź `nodeDB->hasValidPosition()`
   - Sprawdź czy `position.has_latitude_i && position.has_longitude_i`

4. Czy BLE jest połączone:
   - Sprawdź `service->isToPhoneQueueEmpty()`
   - Sprawdź czy aplikacja Android jest połączona

### Problem: Pakiety nie docierają do aplikacji

**Sprawdź:**
1. Czy aplikacja Android obsługuje standardowe pakiety Meshtastic
2. Czy UUID charakterystyki BLE są poprawne
3. Czy aplikacja parsuje pakiety z portu `POSITION_APP`

**Rozwiązanie:**
- Może być potrzebna modyfikacja aplikacji Android
- Zobacz sekcję "Integracja z aplikacją Android" w PODSUMOWANIE_IMPLEMENTACJI.md

### Problem: Zbyt częste wysyłanie

**Rozwiązanie:**
- Zwiększ `sendIntervalMs` w `BleGpsModule.h`:
  ```cpp
  uint32_t sendIntervalMs = 10000; // 10 sekund zamiast 5
  ```

## Krok 7: Integracja z Aplikacją Android

### Format danych:
Moduł wysyła standardowe pakiety Meshtastic protobuf typu `meshtastic_Position`.

### Wymagane modyfikacje (jeśli aplikacja używa uproszczonego formatu):

1. **Parser protobuf:**
   - Dodaj bibliotekę protobuf dla Meshtastic
   - Parsuj pakiety z portu `meshtastic_PortNum_POSITION_APP`

2. **Przykład parsowania:**
   ```kotlin
   // W MeshtasticDataParser.kt
   fun parseMeshtasticPosition(data: ByteArray): LocationData? {
       // Parsuj pełny pakiet Meshtastic
       val packet = MeshPacket.parseFrom(data)
       if (packet.decoded.portnum == PortNum.POSITION_APP) {
           val position = Position.parseFrom(packet.decoded.payload)
           return LocationData(
               nodeId = packet.from.toString(),
               latitude = position.latitudeI * 1e-7,
               longitude = position.longitudeI * 1e-7,
               altitude = position.altitude.toFloat(),
               timestamp = position.time
           )
       }
       return null
   }
   ```

## Checklist Testowania

- [✅] Firmware kompiluje się bez błędów
- [ ] Moduł inicjalizuje się poprawnie (logi)
- [ ] Moduł wysyła pozycję gdy GPS ma lock
- [ ] Moduł nie wysyła gdy GPS nie ma lock
- [ ] Moduł wysyła fixed position bez GPS
- [ ] Pakiety są buforowane gdy BLE nie jest połączone
- [ ] Pakiety docierają do aplikacji Android
- [ ] Interwał wysyłania jest zgodny z konfiguracją (5 sekund)
- [ ] Brak wycieków pamięci
- [ ] Zużycie baterii jest akceptowalne

## Następne Kroki

Po pomyślnym teście:
1. Zoptymalizuj interwał wysyłania jeśli potrzeba
2. Dodaj konfigurację przez AdminMessage (opcjonalnie)
3. Dodaj wysyłanie tylko gdy pozycja się zmieniła (opcjonalnie)
4. Zintegruj z aplikacją Android jeśli potrzebne modyfikacje

## Wsparcie

W razie problemów:
1. Sprawdź logi debug
2. Sprawdź dokumentację Meshtastic Module API
3. Sprawdź PODSUMOWANIE_IMPLEMENTACJI.md
4. Sprawdź kod źródłowy innych modułów (PositionModule, DeviceTelemetryModule)

