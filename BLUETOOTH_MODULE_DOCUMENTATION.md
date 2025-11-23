a# Dokumentacja Modułu Bluetooth Low Energy (BLE)

## Przegląd

Moduł Bluetooth Low Energy (BLE) odpowiada za komunikację z urządzeniami Meshtastic. Zapewnia skanowanie, łączenie i odbieranie danych lokalizacji z węzłów Meshtastic przez protokół BLE.

## Architektura

Moduł BLE składa się z następujących komponentów:

### 1. **MeshtasticBleManager** (`data/ble/MeshtasticBleManager.kt`)
Główny manager odpowiedzialny za zarządzanie komunikacją BLE.

**Odpowiedzialności:**
- Inicjalizacja i zarządzanie `BluetoothAdapter` i `BluetoothLeScanner`
- Skanowanie urządzeń BLE z filtrowaniem Meshtastic
- Łączenie i rozłączanie z urządzeniami
- Zarządzanie połączeniami GATT
- Emisja aktualizacji lokalizacji

**Kluczowe właściwości:**
- `connectionState: StateFlow<Int>` - stan połączenia (BluetoothProfile.STATE_*)
- `availableDevices: StateFlow<List<BleDevice>>` - lista wykrytych urządzeń
- `connectedNodes: StateFlow<List<NodeData>>` - lista połączonych węzłów
- `locationUpdates: Flow<LocationData>` - strumień aktualizacji lokalizacji

**Główne metody:**
- `isBluetoothAvailable(): Boolean` - sprawdza dostępność Bluetooth
- `startScanning()` - rozpoczyna skanowanie urządzeń
- `stopScanning()` - zatrzymuje skanowanie
- `connectToDevice(deviceAddress: String, isUserNode: Boolean): Boolean` - łączy się z urządzeniem
- `disconnect(deviceAddress: String)` - rozłącza się z urządzeniem
- `disconnectAll()` - rozłącza się ze wszystkimi urządzeniami
- `cleanup()` - czyści zasoby

**Szczegóły implementacji:**
- Używa `ScanFilter` z UUID serwisu Meshtastic do filtrowania urządzeń
- Tryb skanowania: `SCAN_MODE_LOW_LATENCY` dla szybkiego wykrywania
- Generuje Node ID z adresu MAC urządzenia (ostatnie 8 znaków bez dwukropków)
- Obsługuje wiele równoczesnych połączeń GATT
- Automatycznie wykrywa serwisy i włącza powiadomienia po połączeniu

### 2. **MeshtasticGattCallback** (`data/ble/MeshtasticGattCallback.kt`)
Callback obsługujący komunikację GATT z urządzeniami Meshtastic.

**Odpowiedzialności:**
- Obsługa zmian stanu połączenia GATT
- Odkrywanie serwisów i charakterystyk
- Włączanie powiadomień dla charakterystyki telemetrycznej
- Odbieranie i parsowanie danych lokalizacji

**Kluczowe właściwości:**
- `connectionState: StateFlow<Int>` - stan połączenia dla tego urządzenia
- `locationUpdates: Channel<LocationData>` - kanał aktualizacji lokalizacji
- `nodeDataUpdates: Channel<NodeData>` - kanał aktualizacji danych węzła

**Obsługiwane callbacki:**
- `onConnectionStateChange()` - zmiana stanu połączenia (CONNECTED/DISCONNECTED)
- `onServicesDiscovered()` - odkrycie serwisów, automatyczne włączenie powiadomień
- `onCharacteristicChanged()` - odbiór danych z charakterystyki telemetrycznej
- `onCharacteristicRead()` - odczyt danych z charakterystyki

**Szczegóły implementacji:**
- Automatycznie włącza powiadomienia po odkryciu serwisów
- Używa `CLIENT_CHARACTERISTIC_CONFIG` descriptor do włączenia powiadomień
- Przekazuje surowe dane do `MeshtasticDataParser` do parsowania

### 3. **MeshtasticDataParser** (`data/ble/MeshtasticDataParser.kt`)
Parser danych z protokołu Meshtastic.

**Odpowiedzialności:**
- Parsowanie surowych danych BLE do obiektów `LocationData`
- Parsowanie danych węzła do obiektów `NodeData`
- Ekstrakcja Node ID z danych

**Format danych (uproszczony):**
```
- Byte 0-1: Node ID (2 bytes)
- Byte 2-5: Latitude (4 bytes, float)
- Byte 6-9: Longitude (4 bytes, float)
- Byte 10-13: Altitude (4 bytes, float, opcjonalne)
- Byte 14-17: Timestamp (4 bytes, uint32)
- Byte 18: Accuracy (1 byte, opcjonalne)
```

**Uwaga:** To jest uproszczony parser. Rzeczywisty protokół Meshtastic używa protobuf.

**Główne metody:**
- `parseLocationData(data: ByteArray, nodeId: String, isUserNode: Boolean): LocationData?`
- `parseNodeData(data: ByteArray, nodeId: String): NodeData?`
- `parseNodeId(data: ByteArray): String?`
- `createLocationRequest(): ByteArray` - tworzy żądanie lokalizacji (do przyszłej implementacji)

### 4. **MeshtasticBleConstants** (`data/ble/MeshtasticBleConstants.kt`)
Stałe UUID dla komunikacji BLE z Meshtastic.

**Definiowane UUID:**
- `MESHTASTIC_SERVICE_UUID`: `6ba1b218-15a8-461f-9f58-069479b0b9d0`
- `TELEMETRY_CHARACTERISTIC_UUID`: `f75c76d2-129e-4dad-a1dd-786f440672e0`
- `TEXT_MESSAGE_CHARACTERISTIC_UUID`: `8ba1b218-15a8-461f-9f58-069479b0b9d0`
- `CONFIG_CHARACTERISTIC_UUID`: `9ba1b218-15a8-461f-9f58-069479b0b9d0`
- `CLIENT_CHARACTERISTIC_CONFIG`: `00002902-0000-1000-8000-00805f9b34fb` (standardowy descriptor)

**Inne stałe:**
- `MESHTASTIC_DEVICE_NAME_PREFIX`: `"Meshtastic"` - prefiks nazwy urządzenia do filtrowania

**Uwaga:** UUID mogą się różnić w zależności od wersji firmware Meshtastic.

### 5. **BleRepository** (`domain/repository/BleRepository.kt`)
Interfejs repozytorium definiujący kontrakt dla komunikacji BLE.

**Definiowane typy:**
- `BleConnectionState` - enum stanów połączenia:
  - `DISCONNECTED` - rozłączony
  - `SCANNING` - skanowanie
  - `CONNECTING` - łączenie
  - `CONNECTED` - połączony
  - `ERROR` - błąd

- `BleDevice` - model urządzenia BLE:
  - `address: String` - adres MAC
  - `name: String?` - nazwa urządzenia
  - `rssi: Int?` - siła sygnału

**Metody interfejsu:**
- `connectionState: StateFlow<BleConnectionState>` - stan połączenia
- `availableDevices: Flow<List<BleDevice>>` - dostępne urządzenia
- `connectedNodes: Flow<List<NodeData>>` - połączone węzły
- `locationUpdates: Flow<LocationData>` - aktualizacje lokalizacji
- `startScanning()` - rozpoczyna skanowanie
- `stopScanning()` - zatrzymuje skanowanie
- `connectToDevice(deviceAddress: String): Result<Unit>` - łączy się z urządzeniem
- `disconnect(deviceAddress: String)` - rozłącza się
- `disconnectAll()` - rozłącza się ze wszystkimi

### 6. **BleRepositoryImpl** (`data/repository/BleRepositoryImpl.kt`)
Implementacja repozytorium BLE.

**Odpowiedzialności:**
- Mapowanie stanów z `MeshtasticBleManager` do `BleConnectionState`
- Obserwowanie aktualizacji lokalizacji z urządzeń
- Przekazywanie lokalizacji do głównego strumienia

**Szczegóły implementacji:**
- Używa `CoroutineScope` z `SupervisorJob` i `Dispatchers.Default`
- Mapuje stany `BluetoothProfile.STATE_*` na `BleConnectionState`
- Obserwuje kanały lokalizacji z każdego urządzenia i przekazuje je do `bleManager.locationUpdates`
- Śledzi stan skanowania (`isScanning`) dla poprawnego mapowania stanów

## Przepływ danych

### 1. Skanowanie urządzeń
```
Użytkownik/ViewModel
    ↓
BleRepository.startScanning()
    ↓
MeshtasticBleManager.startScanning()
    ↓
BluetoothLeScanner.startScan()
    ↓
ScanCallback.onScanResult()
    ↓
Filtrowanie urządzeń Meshtastic
    ↓
BleRepository.availableDevices (Flow)
```

### 2. Łączenie z urządzeniem
```
Użytkownik/ViewModel
    ↓
BleRepository.connectToDevice(address)
    ↓
MeshtasticBleManager.connectToDevice(address)
    ↓
BluetoothDevice.connectGatt()
    ↓
MeshtasticGattCallback.onConnectionStateChange(CONNECTED)
    ↓
MeshtasticGattCallback.onServicesDiscovered()
    ↓
Włączenie powiadomień dla charakterystyki telemetrycznej
    ↓
BleRepository.connectedNodes (Flow)
```

### 3. Odbieranie danych lokalizacji
```
MeshtasticGattCallback.onCharacteristicChanged()
    ↓
MeshtasticDataParser.parseLocationData()
    ↓
MeshtasticGattCallback.locationUpdates (Channel)
    ↓
BleRepositoryImpl.observeDeviceLocationUpdates()
    ↓
MeshtasticBleManager.emitLocationUpdate()
    ↓
BleRepository.locationUpdates (Flow)
    ↓
ViewModel/UI (opcjonalnie: zapis do bazy danych przez LocationRepository)
```

**Uwaga:** Aktualizacje lokalizacji z BLE są emitowane przez `bleRepository.locationUpdates`, ale nie są automatycznie zapisywane do bazy danych. Aby zapisać je do bazy, należy w ViewModel lub innym komponencie obserwować ten Flow i wywołać `locationRepository.saveLocation()` dla każdej aktualizacji.

## Integracja z aplikacją

### Inicjalizacja
Moduł BLE jest inicjalizowany w `MeshTrackerApplication`:

```kotlin
val bleRepository: BleRepository by lazy {
    val bleManager = MeshtasticBleManager(this)
    BleRepositoryImpl(this, bleManager)
}
```

### Użycie w ViewModels

**MapViewModel:**
- Obserwuje `bleRepository.connectionState` i `bleRepository.connectedNodes` dla informacji o połączeniu
- Umożliwia skanowanie (`startBleScanning()`, `stopBleScanning()`) i łączenie z urządzeniami (`connectToDevice()`)
- Dane lokalizacji są odczytywane z bazy danych przez `GetLatestLocationUseCase.getAllLatest()` (nie bezpośrednio z BLE Flow)
- Wyświetla liczbę połączonych węzłów i stan połączenia
- **Uwaga:** Aby zapisywać aktualizacje lokalizacji z BLE do bazy danych, należy dodać obserwację `bleRepository.locationUpdates` i wywołać `locationRepository.saveLocation()` dla każdej aktualizacji

**SettingsViewModel:**
- Zarządza stanem połączenia BLE (`bleRepository.connectionState`)
- Umożliwia rozłączanie wszystkich urządzeń (`disconnectAll()`)
- Wyświetla stan połączenia w interfejsie użytkownika

## Uprawnienia

Moduł wymaga następujących uprawnień (zdefiniowanych w `AndroidManifest.xml`):

**Android 12+ (API 31+):**
- `BLUETOOTH_SCAN` (z flagą `neverForLocation`)
- `BLUETOOTH_CONNECT`

**Android < 12:**
- `BLUETOOTH`
- `BLUETOOTH_ADMIN`

**Dodatkowe:**
- `ACCESS_FINE_LOCATION` - wymagane dla skanowania BLE (nawet z `neverForLocation`)

## Obsługa błędów

### Błędy uprawnień
- Wszystkie operacje BLE są opakowane w `try-catch` dla `SecurityException`
- Błędy są logowane i stan połączenia jest ustawiany na `ERROR` lub `DISCONNECTED`

### Błędy połączenia
- `onScanFailed()` - loguje błąd i ustawia stan na `DISCONNECTED`
- `onServicesDiscovered()` - sprawdza status i loguje błędy
- Nieudane połączenia zwracają `Result.failure()` w `connectToDevice()`

### Błędy parsowania
- `MeshtasticDataParser` zwraca `null` w przypadku błędów parsowania
- Błędy są cicho ignorowane (nie przerywają przepływu)

## Optymalizacje

### Zużycie baterii
- Używa `SCAN_MODE_LOW_LATENCY` tylko podczas aktywnego skanowania
- Skanowanie jest zatrzymywane po połączeniu
- Powiadomienia zamiast ciągłego odczytu (mniejsze zużycie energii)

### Zarządzanie zasobami
- `cleanup()` zamyka wszystkie połączenia i zatrzymuje skanowanie
- Kanały są zamykane w `MeshtasticGattCallback.close()`
- GATT połączenia są zamykane przy rozłączaniu

## Ograniczenia i uwagi

1. **Format danych:** Parser używa uproszczonego formatu. Rzeczywisty protokół Meshtastic używa protobuf i może wymagać biblioteki protobuf do pełnej obsługi.

2. **UUID:** UUID mogą się różnić w zależności od wersji firmware Meshtastic. W razie potrzeby należy je dostosować.

3. **Node ID:** Node ID jest generowane z adresu MAC (ostatnie 8 znaków). Może nie odpowiadać rzeczywistemu Node ID z protokołu Meshtastic.

4. **Wielokrotne połączenia:** Moduł obsługuje wiele równoczesnych połączeń, ale nie testowano z dużą liczbą urządzeń.

5. **Automatyczne ponowne łączenie:** Obecnie nie ma automatycznego ponownego łączenia przy utracie połączenia. Wymaga to dodatkowej implementacji.

## Testowanie

### Scenariusze testowe:
1. Skanowanie urządzeń Meshtastic
2. Łączenie z urządzeniem
3. Odbieranie danych lokalizacji
4. Rozłączanie
5. Obsługa błędów uprawnień
6. Obsługa utraty połączenia
7. Wielokrotne połączenia

### Wymagania do testów:
- Urządzenie z Androidem i obsługą BLE
- Co najmniej jedno urządzenie Meshtastic z włączonym BLE
- Wszystkie wymagane uprawnienia

## Przyszłe ulepszenia

1. **Pełna obsługa protobuf:** Integracja z biblioteką protobuf dla pełnej obsługi protokołu Meshtastic
2. **Automatyczne ponowne łączenie:** Implementacja automatycznego ponownego łączenia przy utracie połączenia
3. **Wysyłanie danych:** Implementacja wysyłania komend i konfiguracji do urządzeń
4. **Obsługa wiadomości tekstowych:** Wykorzystanie `TEXT_MESSAGE_CHARACTERISTIC_UUID`
5. **Cache urządzeń:** Zapisywanie ostatnio używanych urządzeń
6. **Bonding:** Obsługa parowania urządzeń dla lepszej stabilności

## Zależności

Moduł używa tylko standardowych bibliotek Androida:
- `android.bluetooth.*` - API Bluetooth
- `kotlinx.coroutines.*` - Coroutines i Flow
- Brak zewnętrznych bibliotek

## Pliki modułu

```
app/src/main/java/com/example/meshtracker/
├── data/
│   ├── ble/
│   │   ├── MeshtasticBleManager.kt
│   │   ├── MeshtasticGattCallback.kt
│   │   ├── MeshtasticDataParser.kt
│   │   └── MeshtasticBleConstants.kt
│   └── repository/
│       └── BleRepositoryImpl.kt
└── domain/
    └── repository/
        └── BleRepository.kt
```

