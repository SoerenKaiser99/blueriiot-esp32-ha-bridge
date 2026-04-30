# Blue Connect Plus Salt (Gold) → Home Assistant via ESP32 (BLE Bridge)

🇩🇪 **Deutsche Version** | [🇬🇧 English](README.md)

Lies deinen **Blueriiot Blue Connect Plus Salt (Gold)** Pool-Sensor direkt in Home Assistant ein — ohne Blueriiot-Cloud, ohne deren App, ohne Abo. Ein ESP32 dient als lokale BLE-zu-WiFi-Bridge, weckt den Sensor, liest pH / ORP / Temperatur / Salzgehalt / Batterie aus und stellt alles als native HA-Entitäten bereit.

![status](https://img.shields.io/badge/status-stabil-success)
![framework](https://img.shields.io/badge/framework-ESPHome-blue)
![license](https://img.shields.io/badge/license-MIT-green)

---

## Warum das Ganze?

Das Blueriiot-Ökosystem ist geschlossen:

- Sensordaten laufen nur über deren Cloud
- Die offizielle App pollt selten und mit Latenz
- Keine native Home-Assistant-Integration
- Akkulaufzeit leidet unter ständigem BLE-Advertising für Cloud-Sync

Diese Bridge läuft **vollständig lokal**, pollt nach **deinem** Zeitplan (Default: alle 30 Min) und liefert dir 5 native HA-Sensoren plus volle Automatisierungs-Kontrolle.

## Was du bekommst

| Entität | Einheit | Quelle |
|---|---|---|
| `sensor.blueconnect_pool_temperature` | °C | Bytes 1-2 des BLE-Frames |
| `sensor.blueconnect_pool_ph` | pH | Bytes 3-4 |
| `sensor.blueconnect_pool_orp` | mV | Bytes 5-6 |
| `sensor.blueconnect_pool_salinity` | g/L | Bytes 7-8 |
| `sensor.blueconnect_pool_battery` | % | Byte 11 |
| `binary_sensor.blueconnect_pool_status` | verbunden/getrennt | BLE-Link-Status |
| `switch.blueconnect_pool_enable` | löst Reading aus | Manuell / Automation |

---

## Hardware

| Bauteil | Hinweise | Ungefährer Preis |
|---|---|---|
| **Blue Connect Plus Salt (Gold)** | Der Pool-Sensor selbst | 200–250 € |
| **ESP32 DevKit** (jedes gängige Board, z.B. ESP32-WROOM-32) | Muss BLE können — die meisten ESP32-Boards können das | 5–10 € |
| **USB-C/Micro-USB-Kabel** | Für den ersten Flash | – |
| **5V Netzteil** | Dauerstrom für den ESP32 (USB-Ladegerät reicht) | – |

Der ESP32 sollte sich in ~5–10 m Abstand zum Pool befinden, mit Sichtlinie zum schwimmenden Sensor. Wetterfestes Outdoor-Gehäuse empfohlen.

> **Hinweis**: Diese Anleitung ist für **Plus Salt (Gold)** geschrieben. Der einfache Blue Connect Go und andere Varianten nutzen das gleiche BLE-Protokoll, belegen aber andere Bytes für Salinity. pH/ORP/Temperatur-Parsing ist identisch.

---

## Software-Voraussetzungen

- **Home Assistant** (egal welche Variante: HAOS, Container, Supervised)
- **ESPHome Builder Add-on** (oder standalone ESPHome ≥ 2026.1)
- **HACS** mit diesen Frontend-Cards (nur für das Dashboard nötig):
  - [`card-mod`](https://github.com/thomasloven/lovelace-card-mod)
  - [`multiple-entity-row`](https://github.com/benct/lovelace-multiple-entity-row)

---

## Schritt-für-Schritt-Anleitung

### 1. ESPHome-Projekt anlegen

In Home Assistant → **Einstellungen → Add-ons → ESPHome Builder → Web-UI öffnen**.

Klick auf **„+ New Device"** → nenne es `blueconnect` → **Skip** (wir liefern das YAML).

### 2. Secrets konfigurieren

ESPHome speichert geteilte Werte in `/config/esphome/secrets.yaml`. Öffne sie (über den **„Secrets"**-Button oben rechts in der ESPHome-UI) und füge hinzu:

```yaml
wifi_ssid: "DeineWiFiSSID"
wifi_password: "DeinWiFiPasswort"
blueriiot_mac: "00:A0:50:XX:XX:XX"   # wird in Schritt 4 ausgefüllt
blueriiot_nameprefix: "Pool"
blueriiot_idprefix: "pool"
web_server_password: "WähleEinStarkesPasswort"
```

Eine Vorlage liegt unter [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example).

### 3. Erster Flash — MAC-Adresse des Sensors ermitteln

Wir kennen die MAC noch nicht, also flashen wir zuerst eine **Scan-Firmware**.

Kopiere [`esphome/blueconnect-scan.yaml`](esphome/blueconnect-scan.yaml) in dein ESPHome-Projekt (ersetze das automatisch generierte YAML). **Ersetze** diese Platzhalter:

- `REPLACE_WITH_YOUR_API_ENCRYPTION_KEY` → klick *Generate Random Key* in einem Texttool oder nutze ESPHomes Auto-Key
- `REPLACE_WITH_YOUR_OTA_PASSWORD` → beliebiges Passwort für OTA-Updates
- `REPLACE_WITH_FALLBACK_PASSWORD` → Passwort für den Fallback-AP

Klick **Install → Plug into the computer running ESPHome Web Dashboard** → ESP32 per USB anschließen → flashen.

### 4. MAC-Adresse finden

Sobald der ESP32 im WiFi ist, klick **Logs** in ESPHome. Du siehst einen Strom wie:

```
[I][ble_scan]: Found: MAC=00:A0:50:04:4A:18  Name=B200C21365  RSSI=-32
[I][ble_scan]: Found: MAC=FC:23:CD:78:BB:4A  Name=Luba-MNPKVJQ7  RSSI=-86
...
```

Such ein Gerät, das alle drei Kriterien erfüllt:

1. **Name nach Muster `B200xxxxxx`** (Blue Connect Plus Salt sendet mit diesem Prefix)
2. **MAC startet mit `00:A0:50`** — die offiziell registrierte OUI von Blue Riiot Labs
3. **Starkes RSSI** (−40 oder besser, wenn du direkt am Pool stehst)

Kopiere diese MAC in `secrets.yaml` als `blueriiot_mac`.

### 5. Produktiv-Firmware flashen

Ersetze das YAML deines Projekts durch [`esphome/blueconnect.yaml`](esphome/blueconnect.yaml). Ersetze die gleichen drei Platzhalter wie in Schritt 3 (verwende **dieselben** Werte — Keys müssen zwischen Flashes übereinstimmen, sonst funktioniert OTA nicht).

Klick **Install → Wirelessly**.

### 6. Gerät zu Home Assistant hinzufügen

HA findet ESPHome-Geräte normalerweise automatisch via mDNS. Falls nicht:

**Einstellungen → Geräte & Dienste → + Integration hinzufügen → ESPHome**
- Host: `blueconnect.local` (oder die IP)
- Encryption Key: das, was du unter `api → encryption → key` eingetragen hast

Du siehst jetzt alle oben genannten Entitäten.

### 7. Sensor zum ersten Mal aufwecken

Der Blue Connect Plus Salt schläft die meiste Zeit. Für das erste Reading:

1. `switch.blueconnect_pool_enable` in HA auf **on** schalten
2. **Logs** in ESPHome beobachten — du solltest sehen:
   ```
   Connected to Blueriiot sensor
   raw_hex: 33.E0.06.19.08.25.0B.73.00.FF.0D.10
   temp: 17.60
   ph: 6.89
   orp: 717.5
   salt: 6.4
   bat: 44.4
   ```
3. Der Switch geht nach erfolgreichem Reading **automatisch wieder aus**
4. Alle fünf Sensor-Entitäten haben jetzt Werte

Wenn nichts passiert (~30 Sekunden warten): **Blueriiot-Sonde kurz ins Wasser tauchen** oder die goldenen Kontakte unten kurzschließen. Der Sensor sendet nur Advertisements, wenn er „aktiv" ist — und die Verbindung braucht ankommende Advertisements.

### 8. Periodisches Auslesen automatisieren

Kopiere [`homeassistant/automation.yaml`](homeassistant/automation.yaml) in **Einstellungen → Automatisierungen → ⋮ → In YAML bearbeiten**. Default ist alle 30 Minuten — ändere den Trigger, falls du andere Intervalle willst.

> **Warum alle 30 Min?** Die Knopfzelle des Blue Connect hält bei dieser Frequenz ~12 Monate. Polling im Minutentakt killt die Batterie in Wochen.

### 9. (Optional) Dashboard-Card installieren

1. Falls noch nicht vorhanden: **HACS** installieren, dann `card-mod` und `multiple-entity-row` aus HACS Frontend installieren.
2. Helper-Toggle anlegen: **Einstellungen → Geräte & Dienste → Helfer → + Helfer erstellen → Schalter** mit Name `blueriiot_tipps`.
3. Beliebiges Dashboard öffnen → **Bearbeiten → Karte hinzufügen → Manuell** → Inhalt von [`homeassistant/dashboard-card.yaml`](homeassistant/dashboard-card.yaml) reinkopieren.

Die Card zeigt Live-Werte, farbcodierte Bereiche, Dosier-Empfehlungen und ausklappbare Hilfe-Notizen.

---

## Kalibrierung

Die Salinity-Formel nutzt Divisor `/18.0`, kalibriert auf die offizielle Blueriiot-App bei einem Blue Connect Plus Salt. Falls dein Wert abweicht, so kalibrierst du nach:

1. Mit der **offiziellen Blueriiot-App** ein Reading machen — Salzgehalt notieren (z.B. `6.4 g/L`)
2. Gleichzeitig in den ESPHome-**Logs** nach der `raw_hex`-Zeile suchen — Bytes 7-8 herauspicken
3. Rohwert berechnen: `raw = (byte8 << 8) + byte7`
4. Neuer Divisor: `raw / app_wert`

**Beispiel:**
```
raw_hex: 33.E0.06.19.08.25.0B.73.00.FF.0D.10
                            ^^ ^^
                          byte7 byte8
raw = (0x00 << 8) + 0x73 = 115
App zeigt: 6,4 g/L
Divisor = 115 / 6.4 = 17.97
```

Dann in `blueconnect.yaml` die Lambda-Zeile anpassen:
```cpp
float salt = (float)((int16_t)(x[8]<<8) + x[7]) / 17.97;
```

> **Hinweis**: Sensor-Readings haben einen natürlichen Jitter von ±0.1 g/L. Lass dich nicht auf Nachkommastellen ein — Pool-Salzgehalt ändert sich über Wochen, nicht Minuten.

---

## So funktioniert das BLE-Protokoll

Jede Notification ist **12 Bytes**, little-endian, wird vom Sensor gepusht, sobald wir `0x01` auf die Trigger-Characteristic schreiben:

| Byte(s) | Bedeutung | Formel | Beispiel (`33.E0.06.19.08.25.0B.73.00.FF.0D.10`) |
|---|---|---|---|
| 0 | Header / Status | – | `0x33` |
| 1-2 | Temperatur | `raw / 100` | `0x06E0 = 1760 → 17,60 °C` |
| 3-4 | pH (raw) | `(2048 - raw) / 232 + 7` | `0x0819 = 2073 → 6,89` |
| 5-6 | ORP (raw) | `raw / 3.86 - 21,58` | `0x0B25 = 2853 → 717,5 mV` |
| 7-8 | Salzgehalt (raw) | `raw / 18.0` (kalibriert) | `0x0073 = 115 → 6,4 g/L` |
| 9-10 | Unbekannt / reserviert | – | `0x0DFF` |
| 11 | Batterie | `raw / 36 * 100` | `0x10 = 16 → 44,4 %` |

BLE-Service-/Characteristic-UUIDs:

- **Service**: `F3300001-F0A2-9B06-0C59-1BC4763B5C00`
- **Trigger** (wir schreiben `0x01`): `F3300002-F0A2-9B06-0C59-1BC4763B5C00`
- **Notify** (wir lesen): `F3300003-F0A2-9B06-0C59-1BC4763B5C00`

---

## Troubleshooting

### „Secret 'blueriiot_mac' not defined"

Deiner `secrets.yaml` fehlen Einträge. Siehe Schritt 2.

### „Error resolving IP address of blueconnect.local"

mDNS funktioniert nicht oder der ESP ist noch nicht im WiFi. In der DHCP-Tabelle deines Routers nach der zugewiesenen IP suchen, dann entweder:

- Die IP direkt in ESPHomes *Install → Manual IP* nutzen
- Oder eine statische IP unter `wifi:` im YAML eintragen:
  ```yaml
  wifi:
    manual_ip:
      static_ip: 192.168.X.XXX
      gateway: 192.168.X.1
      subnet: 255.255.255.0
  ```

### „Missing toolchain directory 'None'" beim Compile

PlatformIOs Toolchain-Cache ist kaputt. Löschen und neu kompilieren:

```bash
# Im ESPHome-Add-on-Container:
rm -rf /data/cache/platformio /root/.platformio/tools/toolchain-xtensa-esp-elf
```

Dann das ESPHome-Add-on neu starten und nochmal compilen.

### Sensor verbindet, aber Werte bleiben leer

Die Blueriiot-Sonde muss **aktiv** sein (im Wasser oder Kontakte kurzgeschlossen). Trocken gelagerte Sensoren reagieren erst nach physischem Aufwecken. Sobald ein Reading durch ist, funktionieren die nächsten ~5 Minuten auch außerhalb des Wassers.

### Salzgehalt-Wert ist um einen Faktor daneben

Siehe **Kalibrierung** oben — Divisor neu aus aktuellem raw_hex-Log ableiten.

### HA zeigt weniger Nachkommastellen als ESPHome

HA cached die Display-Precision der Entität ab erster Registrierung. Fix:

**Einstellungen → Geräte & Dienste → Blueconnect → [Sensor] → ⚙ → Anzeigegenauigkeit**

---

## Datei-Übersicht

```
.
├── esphome/
│   ├── blueconnect-scan.yaml      # Schritt 3: MAC ermitteln
│   ├── blueconnect.yaml           # Schritt 5: Produktiv-Firmware
│   └── secrets.yaml.example       # Vorlage für /config/esphome/secrets.yaml
├── homeassistant/
│   ├── automation.yaml            # Schritt 8: 30-Min-Polling
│   ├── input_boolean.yaml         # Helper-Toggle für Dashboard
│   └── dashboard-card.yaml        # Schritt 9: Komplette Lovelace-Card
├── LICENSE
├── README.md                      # English
└── README.de.md                   # Deutsch
```

---

## Credits & verwandte Projekte

- BLE-Protokoll-Reverse-Engineering: ursprüngliche Arbeit von Community-Mitgliedern im [ESPHome-Forum](https://community.home-assistant.io/) und dem [Blueriiot-Python-Projekt](https://github.com/wjtje/blueriiot-ble).
- Diese Implementierung nutzt ESPHomes nativen `ble_client` und das `text_sensor`-notify-Pattern.

---

## Lizenz

MIT — siehe [LICENSE](LICENSE).
