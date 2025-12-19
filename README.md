# Sentri ESP32 Access Control Device

A robust ESP32-based access control system with dual RFID readers, Firebase Firestore integration, and Over-The-Air (OTA) update capabilities.

## Features

### Core Functionality
- **Dual RFID Readers** - Entry and Exit MFRC522 readers for bidirectional access control
- **Firebase Firestore Integration** - Real-time card authorization and access logging
- **Relay Control** - Non-blocking door unlock pulse mechanism
- **Time-Based Access** - Shift scheduling with start/end times per card

### Reliability & Recovery
- **Wi-Fi Auto-Recovery** - Automatic reconnection with fallback provisioning portal
- **Offline Operation** - Local card cache for continued operation during network outages
- **Watchdog Timer** - Automatic restart on device hang
- **Memory Monitoring** - Heap health checks with emergency flush on low memory
- **SPIFFS Persistence** - Configuration, logs, and card data survive reboots

### Remote Management
- **OTA Updates** - Push firmware updates remotely without physical access
- **HTTP Status API** - Real-time device diagnostics via `/status` endpoint
- **Factory Reset** - Hardware button for config reset (short press: restart, long press: wipe config)

### Security
- **Provisioning Portal** - Secure AP mode for initial device setup
- **Card Debouncing** - Prevents double-reads and replay attacks
- **Thread-Safe Operations** - Mutex-protected data structures

---

## Hardware Requirements

### Components
| Component | Quantity | Pin Connections |
|-----------|----------|-----------------|
| ESP32 DevKit | 1 | - |
| MFRC522 RFID Reader (Entry) | 1 | SS=5, RST=21 |
| MFRC522 RFID Reader (Exit) | 1 | SS=13, RST=15 |
| 5V Relay Module | 1 | GPIO 32 |
| Reset Button (Active HIGH) | 1 | GPIO 33 |
| Technician Jumper (Optional) | 1 | GPIO 25 |

### Pin Configuration
```cpp
#define MFRC522_ENTRY_SS   5
#define MFRC522_ENTRY_RST  21
#define MFRC522_EXIT_SS    13
#define MFRC522_EXIT_RST   15
#define FACTORY_RESET_PIN  33
#define RELAY              32
#define FACTORY_RESET_FULL_PIN 25  // Optional technician wipe
```

### Wiring Diagram
```
ESP32          MFRC522 Entry    MFRC522 Exit
------         ------------     -----------
3.3V    ────── VCC              VCC
GND     ────── GND              GND
GPIO 5  ────── SDA (SS)
GPIO 21 ────── RST
GPIO 13 ──────────────────────  SDA (SS)
GPIO 15 ──────────────────────  RST
GPIO 18 ────── SCK              SCK
GPIO 19 ────── MISO             MISO
GPIO 23 ────── MOSI             MOSI

GPIO 32 ────── Relay IN
GPIO 33 ────── Reset Button (to 3.3V when pressed)
GPIO 25 ────── Technician Jumper (to GND for full wipe)
```

---

## Software Setup

### Prerequisites
- Arduino IDE 2.x or PlatformIO
- ESP32 Board Support Package
- Required Libraries:
  - `FirebaseClient` (by mobizt)
  - `MFRC522` (by miguelbalboa)

### Installation

1. **Clone/Download the code**

2. **Install Libraries** (Arduino IDE):
   - Sketch → Include Library → Manage Libraries
   - Search and install: `FirebaseClient`, `MFRC522`

3. **Select Board**:
   - Tools → Board → ESP32 Dev Module
   - Tools → Partition Scheme → "Default 4MB with spiffs" or "Minimal SPIFFS (1.9MB APP with OTA)"

4. **Flash the firmware**:
   - Connect ESP32 via USB
   - Click Upload

---

## Firebase Setup

### 1. Create Firebase Project
1. Go to [Firebase Console](https://console.firebase.google.com)
2. Create new project or use existing
3. Enable **Firestore Database**
4. Enable **Authentication** → Email/Password provider

### 2. Create Service User
1. Authentication → Users → Add User
2. Create a dedicated device user (e.g., `device@yourdomain.com`)
3. Note the email and password

### 3. Get Project Credentials
- **API Key**: Project Settings → General → Web API Key
- **Project ID**: Project Settings → General → Project ID

### 4. Firestore Structure

```
├── devices/
│   └── {MAC_ADDRESS}/
│       ├── device_id: "AA:BB:CC:DD:EE:FF"
│       ├── business_id: "business123"
│       ├── device_status: "online"
│       └── device_last_online: timestamp
│
├── businessess/
│   └── {business_id}/
│       ├── business_devices/
│       │   └── {MAC_ADDRESS}/
│       │       ├── device_id: "AA:BB:CC:DD:EE:FF"
│       │       ├── device_status: true/false
│       │       ├── device_name: "Main Gate"
│       │       └── device_last_online: timestamp
│       │
│       ├── active_cards/
│       │   └── active_cards/
│       │       ├── updated_at: timestamp
│       │       └── cards: [
│       │           {
│       │             card_data: "A1B2C3D4",
│       │             start_time: "09:00",
│       │             end_time: "18:00"
│       │           }
│       │         ]
│       │
│       ├── access_logs/
│       │   └── {auto_generated_id}/
│       │       ├── log_card_data: "A1B2C3D4"
│       │       ├── log_device_id: "AA:BB:CC:DD:EE:FF"
│       │       ├── log_status: "granted" | "denied"
│       │       ├── log_type: "entry" | "exit"
│       │       └── log_timestamp: timestamp
│       │
│       └── cards/
│           └── {card_id}/
│               ├── card_status: true/false
│               ├── card_assigned_to: "user123"
│               └── card_assigned_type: "employee"
│
└── firmware_updates/
    └── latest/
        ├── version: "1.0.1"
        ├── download_url: "https://..."
        ├── mandatory: false
        └── release_notes: "Bug fixes"
```

### 5. Firestore Security Rules
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Devices can read/write their own documents
    match /devices/{deviceId} {
      allow read, write: if request.auth != null;
    }
    
    match /businessess/{businessId}/{document=**} {
      allow read, write: if request.auth != null;
    }
    
    match /firmware_updates/{document=**} {
      allow read: if request.auth != null;
      allow write: if false; // Admin only via console
    }
  }
}
```

---

## Device Provisioning

### First Boot / Factory Reset

1. **Power on the device** - LED should indicate it's searching for WiFi

2. **Connect to Setup AP**:
   - SSID: `Sentri-Setup`
   - Password: `12345678`

3. **Open Configuration Portal**:
   - Navigate to `http://192.168.4.1`

4. **Enter Configuration**:
   | Field | Description |
   |-------|-------------|
   | WiFi SSID | Your network name |
   | WiFi Password | Your network password |
   | API Key | Firebase Web API Key |
   | Project ID | Firebase Project ID |
   | User Email | Device service account email |
   | User Password | Device service account password |
   | Business ID | Your business identifier |
   | Device Name | Friendly name (e.g., "Main Gate") |

5. **Save & Reboot** - Device will connect and register with Firebase

---

## HTTP API Endpoints

When the device is connected to WiFi, these endpoints are available:

### GET /status
Returns device diagnostics in JSON format.

**Response:**
```json
{
  "mac": "AA:BB:CC:DD:EE:FF",
  "firmware_version": "1.0.0",
  "build_date": "Dec 19 2024 08:21:00",
  "wifi_status": 3,
  "ip": "192.168.1.100",
  "rssi": -45,
  "firebase_ready": true,
  "active_cards_loaded": 25,
  "active_set_size": 25,
  "activity_enabled": true,
  "last_heartbeat_ms": 3600000,
  "last_flush_ms": 3600000,
  "firebase_fail_count": 0,
  "ota_update_available": false,
  "available_version": "",
  "free_heap": 150000
}
```

### GET /ota/check
Force check for firmware updates.

**Response:**
```json
{
  "current_version": "1.0.0",
  "update_available": true,
  "available_version": "1.0.1",
  "in_progress": false
}
```

### POST /ota/update
Trigger OTA update (if available).

**Response:** HTML page confirming update started.

### GET /reboot
Restart the device.

---

## OTA (Over-The-Air) Updates

### How It Works
1. Device checks Firestore `firmware_updates/latest` every hour
2. Compares remote version with current version
3. If different, marks update as available
4. If `mandatory: true`, auto-installs immediately
5. Otherwise, waits for manual trigger via `/ota/update`

### Publishing an Update

#### Step 1: Update Version
```cpp
// In Device.ino
#define FIRMWARE_VERSION "1.0.1"  // Increment this
```

#### Step 2: Compile Firmware
**Arduino IDE:**
- Sketch → Export Compiled Binary
- Find `.bin` file in sketch folder

**PlatformIO:**
```bash
pio run
# Binary at: .pio/build/esp32dev/firmware.bin
```

#### Step 3: Host Firmware Binary
Upload `.bin` to an accessible server:
- GitHub Releases
- Firebase Storage (with public URL)
- AWS S3
- Any HTTP server

#### Step 4: Update Firestore
Create/update document at `firmware_updates/latest`:
```json
{
  "version": "1.0.1",
  "download_url": "https://your-server.com/firmware_v1.0.1.bin",
  "mandatory": false,
  "release_notes": "Fixed RFID debouncing issue"
}
```

#### Step 5: Trigger Update
**Option A:** Wait for hourly check

**Option B:** Force check via HTTP:
```bash
curl http://192.168.1.100/ota/check
curl -X POST http://192.168.1.100/ota/update
```

**Option C:** Set `mandatory: true` for immediate auto-update

### OTA Serial Output
```
OTA: Checking for firmware updates...
OTA: Current=1.0.0 Remote=1.0.1
OTA: Update available! Version 1.0.1
OTA: Starting update from https://...
OTA: Firmware size: 1234567 bytes
OTA Progress: 25% (308641/1234567 bytes)
OTA Progress: 50% (617283/1234567 bytes)
OTA Progress: 75% (925925/1234567 bytes)
OTA Progress: 100% (1234567/1234567 bytes)
OTA: Write complete
OTA: Update successful! Rebooting...
```

---

## Factory Reset

### Short Press (1-5 seconds)
- Soft restart
- Configuration preserved

### Long Press (5+ seconds)
- Removes `/config.json`
- Device enters provisioning mode on next boot
- Access logs and card cache preserved

### Full Wipe (Long Press + Technician Jumper)
- Hold reset button for 5+ seconds
- **AND** short GPIO 25 to GND
- Removes ALL data:
  - Configuration
  - Access logs queue
  - Card cache
  - Activity state

---

## Configuration Files (SPIFFS)

| File | Purpose |
|------|---------|
| `/config.json` | WiFi and Firebase credentials |
| `/access_list.json` | Cached active cards with schedules |
| `/access_meta.json` | Last sync timestamp for active cards |
| `/access_logs.ndjson` | Queued access logs (pending upload) |
| `/cards_cache.ndjson` | Individual card status cache |
| `/activity_state.txt` | Device enabled/disabled state |
| `/log_counter.txt` | Sequential log counter |
| `/firmware_meta.json` | Current firmware version metadata |

---

## Timing & Thresholds

| Parameter | Value | Description |
|-----------|-------|-------------|
| `DOOR_PULSE_MS` | 3000ms | Unlock duration |
| `RFID_DEBOUNCE_MS` | 2000ms | Min time between same card scans |
| `HEARTBEAT_MS` | 1 hour | Device status update interval |
| `FLUSH_INTERVAL_MS` | 1 minute | Log flush check interval |
| `OTA_CHECK_INTERVAL_MS` | 1 hour | Firmware update check interval |
| `WIFI_LOOKUP_TIMEOUT_MS` | 2 minutes | WiFi recovery before opening AP |
| `PROVISION_WINDOW_MS` | 5 minutes | Provisioning portal open duration |
| `CARD_CACHE_TTL_SEC` | 24 hours | Card status cache validity |
| `WATCHDOG_TIMEOUT_SEC` | 30 seconds | Watchdog timer timeout |

---

## Troubleshooting

### Device Won't Connect to WiFi
1. Check SSID/password in config
2. Ensure 2.4GHz network (ESP32 doesn't support 5GHz)
3. Move device closer to router
4. Factory reset and re-provision

### Firebase Authentication Fails
1. Verify API Key is correct
2. Check user email/password
3. Ensure Email/Password auth is enabled in Firebase
4. Check Firestore rules allow access

### RFID Reader Not Detecting Cards
1. Check wiring (especially SS and RST pins)
2. Verify 3.3V power supply
3. Check serial output for version register:
   - `0x92` = MFRC522 v2.0 (OK)
   - `0x91` = MFRC522 v1.0 (OK)
   - `0x00` or `0xFF` = Not responding

### OTA Update Fails
| Error | Solution |
|-------|----------|
| "Not enough space" | Use partition scheme with OTA support |
| "HTTP GET failed, code=404" | Verify download URL is accessible |
| "Invalid content length" | Check server returns Content-Length header |
| "Update.end() failed" | Binary may be corrupted, rebuild |

### Device Keeps Restarting
1. Check power supply (stable 5V, 500mA+)
2. Monitor serial for crash dump
3. Check heap memory (should be >20KB free)
4. Watchdog may be triggering - check for blocking operations

---

## Serial Monitor Output

### Normal Boot Sequence
```
=== Device Boot Starting ===
Watchdog initialized: 30 second timeout
Mounting SPIFFS...
SPIFFS mounted (normal).
SPIFFS: used=12345 total=1441792
Booting... heap=280000

=== LOADED CONFIG ===
ssid: 'MyNetwork'
pass length: 10
api_key: 'AIzaSy...'
project_id: 'my-project'
...
=======================

MFRC522 Entry version: 0x92
MFRC522 Exit version: 0x92
MFRC522 (entry & exit) initialized.

Device ID: AA:BB:CC:DD:EE:FF

=== FIRMWARE INFO ===
Version: 1.0.0
Build Date: Dec 19 2024 08:21:00
=====================

WiFi: connecting to MyNetwork... connected
WiFi IP: 192.168.1.100 RSSI=-45
Starting non-blocking time sync (SNTP)...
Starting Firebase init (non-blocking start).
Firebase init completed (app.ready()).
Fetching active_cards from Firestore...
Loaded 25 active cards

=== Setup Complete ===
Free heap: 150000 bytes
```

### Card Scan Events
```
ENTRY granted  card=A1B2C3D4  at=2024-12-19T08:30:00Z  (device_status=ON)
Relay: pulse started (non-blocking)
Relay: pulse ended, locked

EXIT granted  card=A1B2C3D4  at=2024-12-19T17:30:00Z  (device_status=ON)
```

---

## Version History

### v1.0.0 (Current)
- Initial release
- Dual MFRC522 support
- Firebase Firestore integration
- OTA update support
- Non-blocking operations
- Watchdog timer
- Memory monitoring
- WiFi auto-recovery
- Provisioning portal
- Thread-safe card operations

---

## License

MIT License - See LICENSE file for details.

## Support

For issues or feature requests, please open an issue in the repository.
