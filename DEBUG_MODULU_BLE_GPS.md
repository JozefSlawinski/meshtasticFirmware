# Debugowanie Modułu BLE GPS

## Problem: Moduł nie jest widoczny na sensecap-t1000-e

### Krok 1: Sprawdź czy moduł jest kompilowany

Sprawdź w logach kompilacji czy plik `BleGpsModule.cpp` jest kompilowany:

```bash
pio run -e tracker-t1000-e -v
```

Szukaj w logach:
```
Compiling .pio\build\tracker-t1000-e\src\modules\BleGpsModule.cpp.o
```

Jeśli nie widzisz tego pliku, moduł nie jest kompilowany.

### Krok 2: Sprawdź warunki kompilacji

Moduł jest kompilowany tylko gdy:
- `!MESHTASTIC_EXCLUDE_GPS` - GPS nie jest wykluczony
- `!MESHTASTIC_EXCLUDE_BLUETOOTH` - Bluetooth nie jest wykluczony

Sprawdź w `variants/nrf52840/tracker-t1000-e/platformio.ini` czy nie ma:
- `-DMESHTASTIC_EXCLUDE_GPS=1`
- `-DMESHTASTIC_EXCLUDE_BLUETOOTH=1`

### Krok 3: Sprawdź logi podczas startu

Po wgraniu firmware, połącz się przez Serial (115200 baud) i szukaj:

```
BleGpsModule initialized - will send position to phone every 5000 ms
```

Jeśli nie widzisz tego komunikatu:
1. Moduł nie został zainicjalizowany
2. Warunki kompilacji nie są spełnione
3. Moduł nie jest rejestrowany w `setupModules()`

### Krok 4: Sprawdź czy moduł jest rejestrowany

Sprawdź w `src/modules/Modules.cpp` linia 134-136:

```cpp
#if !MESHTASTIC_EXCLUDE_GPS && !MESHTASTIC_EXCLUDE_BLUETOOTH
    bleGpsModule = new BleGpsModule();
#endif
```

### Krok 5: Weryfikacja dla tracker-t1000-e

Dla **tracker-t1000-e** (nRF52):
- ✅ Ma GPS: `HAS_GPS 1` w `variants/nrf52840/tracker-t1000-e/variant.h`
- ✅ Ma Bluetooth: `connectivity: ["bluetooth"]` w `boards/tracker-t1000-e.json`
- ✅ Nie ma `MESHTASTIC_EXCLUDE_BLUETOOTH` w `platformio.ini`

**Moduł powinien być kompilowany i działać!**

### Krok 6: Sprawdź logi działania

Po starcie, moduł powinien logować:

**Gdy GPS ma lock:**
```
BleGpsModule: Using position from GPS object
BleGpsModule: Sent position to phone - lat=..., lon=..., time=...
```

**Gdy GPS nie ma lock:**
```
BleGpsModule: No GPS lock, skipping position send
```

### Krok 7: Debug - Dodaj więcej logów

Jeśli moduł się nie inicjalizuje, dodaj log w konstruktorze:

```cpp
BleGpsModule::BleGpsModule()
    : ProtobufModule("blegps", meshtastic_PortNum_POSITION_APP, &meshtastic_Position_msg),
      concurrency::OSThread("BleGpsModule")
{
    LOG_INFO("=== BleGpsModule CONSTRUCTOR CALLED ===");
    setIntervalFromNow(setStartDelay());
    LOG_INFO("BleGpsModule initialized - will send position to phone every %d ms", sendIntervalMs);
}
```

### Krok 8: Sprawdź czy moduł jest włączony w konfiguracji

Sprawdź czy w konfiguracji urządzenia:
- GPS jest włączony
- Bluetooth jest włączony

Możesz sprawdzić przez aplikację Meshtastic lub AdminMessage.

### Krok 9: Test kompilacji z verbose

```bash
pio run -e tracker-t1000-e -v 2>&1 | grep -i "blegps\|BleGps"
```

Powinieneś zobaczyć:
- Kompilację pliku
- Linkowanie modułu

### Krok 10: Sprawdź symbol w mapie pamięci

Po kompilacji, sprawdź czy symbol `bleGpsModule` jest w mapie:

```bash
pio run -e tracker-t1000-e -t map 2>&1 | grep -i "blegps"
```

## Rozwiązania problemów

### Problem: Moduł nie kompiluje się

**Rozwiązanie:**
1. Sprawdź czy warunki `#if !MESHTASTIC_EXCLUDE_GPS && !MESHTASTIC_EXCLUDE_BLUETOOTH` są spełnione
2. Sprawdź czy plik `BleGpsModule.cpp` jest w katalogu `src/modules/`
3. Sprawdź czy nie ma błędów składniowych

### Problem: Moduł kompiluje się, ale nie inicjalizuje

**Rozwiązanie:**
1. Sprawdź czy `setupModules()` jest wywoływane
2. Sprawdź logi podczas startu
3. Dodaj więcej logów w konstruktorze

### Problem: Moduł inicjalizuje się, ale nie wysyła pozycji

**Rozwiązanie:**
1. Sprawdź czy GPS ma lock: `gpsStatus->getHasLock()`
2. Sprawdź czy pozycja jest ważna: `nodeDB->hasValidPosition()`
3. Sprawdź logi debug

### Problem: Nie widzę logów w ogóle

**Rozwiązanie:**
1. Sprawdź poziom logowania (może być ustawiony na ERROR tylko)
2. Użyj Serial monitor z odpowiednim poziomem logowania
3. Sprawdź czy logi są przekierowane do Serial

## Test szybki

Dodaj tymczasowo w `runOnce()`:

```cpp
int32_t BleGpsModule::runOnce()
{
    LOG_INFO("=== BleGpsModule::runOnce() CALLED ===");
    // ... reszta kodu
}
```

Jeśli widzisz ten log co 5 sekund, moduł działa!

## Kontakt

Jeśli problem nadal występuje:
1. Sprawdź pełne logi kompilacji
2. Sprawdź logi Serial podczas startu
3. Sprawdź czy inne moduły (np. PositionModule) działają

