# Blue Connect Plus Salt (Gold) → Home Assistant via ESP32 (BLE Bridge)

Read your **Blueriiot Blue Connect Plus Salt (Gold)** pool sensor directly into Home Assistant — without the Blueriiot cloud, without their app, without any subscription. An ESP32 acts as a local BLE-to-WiFi bridge that wakes the sensor, reads pH / ORP / temperature / salinity / battery, and exposes everything as native HA entities.

![status](https://img.shields.io/badge/status-working-success)
![framework](https://img.shields.io/badge/framework-ESPHome-blue)
![license](https://img.shields.io/badge/license-MIT-green)

---

## Why this exists

The Blueriiot ecosystem is closed:

- Sensor data only flows through their cloud
- The official app polls infrequently and adds latency
- No native Home Assistant integration
- Battery life suffers from constant BLE advertising for cloud sync

This bridge runs **fully local**, polls on **your** schedule (default: every 30 min), and gives you 5 native HA sensors plus full automation control.

## What you get

| Entity | Unit | Source |
|---|---|---|
| `sensor.blueconnect_pool_temperature` | °C | Bytes 1-2 of BLE frame |
| `sensor.blueconnect_pool_ph` | pH | Bytes 3-4 |
| `sensor.blueconnect_pool_orp` | mV | Bytes 5-6 |
| `sensor.blueconnect_pool_salinity` | g/L | Bytes 7-8 |
| `sensor.blueconnect_pool_battery` | % | Byte 11 |
| `binary_sensor.blueconnect_pool_status` | connected/disconnected | BLE link state |
| `switch.blueconnect_pool_enable` | trigger reading | Manual / automation |

---

## Hardware

| Item | Notes | Approx. price |
|---|---|---|
| **Blue Connect Plus Salt (Gold)** | The pool sensor itself | €200–250 |
| **ESP32 DevKit** (any common board, e.g. ESP32-WROOM-32) | Must support BLE — most ESP32 boards do | €5–10 |
| **USB-C/Micro-USB cable** | For initial flashing | – |
| **5V power supply** | Permanent power for the ESP32 (USB charger works) | – |

The ESP32 needs to be within ~5–10 m of the pool with line-of-sight to the floating sensor. Outdoor weatherproof enclosure recommended.

> **Note**: This guide is written for **Plus Salt (Gold)**. The plain Blue Connect Go and other variants use the same BLE protocol but expose different bytes for salinity. The pH/ORP/temperature parsing is identical.

---

## Software prerequisites

- **Home Assistant** (any flavor: HAOS, Container, Supervised)
- **ESPHome Builder add-on** (or standalone ESPHome ≥ 2026.1)
- **HACS** with these frontend cards (only needed for the dashboard):
  - [`card-mod`](https://github.com/thomasloven/lovelace-card-mod)
  - [`multiple-entity-row`](https://github.com/benct/lovelace-multiple-entity-row)

---

## Step-by-step setup

### 1. Create the ESPHome project

In Home Assistant → **Settings → Add-ons → ESPHome Builder → Open Web UI**.

Click **"+ New Device"** → name it `blueconnect` → **Skip** (we'll provide the YAML).

### 2. Configure secrets

ESPHome stores shared values in `/config/esphome/secrets.yaml`. Open it (via the ESPHome UI's **"Secrets"** button at the top right) and add:

```yaml
wifi_ssid: "YourWiFiSSID"
wifi_password: "YourWiFiPassword"
blueriiot_mac: "00:A0:50:XX:XX:XX"   # filled in step 4
blueriiot_nameprefix: "Pool"
blueriiot_idprefix: "pool"
web_server_password: "ChooseAStrongPassword"
```

A template lives at [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example).

### 3. First flash — discover the sensor's MAC address

We don't know the MAC yet, so we flash a **scan-only firmware** first.

Copy [`esphome/blueconnect-scan.yaml`](esphome/blueconnect-scan.yaml) into your ESPHome project (overwrite the auto-generated YAML). **Replace** these placeholders:

- `REPLACE_WITH_YOUR_API_ENCRYPTION_KEY` → click *Generate Random Key* in any text generator, or use ESPHome's auto-key
- `REPLACE_WITH_YOUR_OTA_PASSWORD` → any password for OTA updates
- `REPLACE_WITH_FALLBACK_PASSWORD` → password for the captive-portal fallback AP

Click **Install → Plug into the computer running ESPHome Web Dashboard** → connect ESP32 via USB → flash.

### 4. Find the MAC address

Once the ESP32 is on WiFi, click **Logs** in ESPHome. You'll see a stream like:

```
[I][ble_scan]: Found: MAC=00:A0:50:04:4A:18  Name=B200C21365  RSSI=-32
[I][ble_scan]: Found: MAC=FC:23:CD:78:BB:4A  Name=Luba-MNPKVJQ7  RSSI=-86
...
```

Look for a device matching all of these:

1. **Name pattern `B200xxxxxx`** (Blue Connect Plus Salt advertises with this prefix)
2. **MAC starting with `00:A0:50`** — Blue Riiot Labs' OUI (officially registered)
3. **Strong RSSI** (−40 or better if you're standing next to the pool)

Copy that MAC and put it into `secrets.yaml` as `blueriiot_mac`.

### 5. Flash the production firmware

Replace your project's YAML with [`esphome/blueconnect.yaml`](esphome/blueconnect.yaml). Replace the same three placeholders as in step 3 (use the **same** values — keys must match between flashes for OTA to work).

Click **Install → Wirelessly**.

### 6. Add the device to Home Assistant

HA usually auto-detects ESPHome devices via mDNS. If not:

**Settings → Devices & Services → + Add Integration → ESPHome**
- Host: `blueconnect.local` (or the IP)
- Encryption key: whatever you put under `api → encryption → key`

You'll now see all entities listed above.

### 7. Wake the sensor for the first time

The Blue Connect Plus Salt is asleep most of the time. To get the first reading:

1. Toggle `switch.blueconnect_pool_enable` **on** in HA
2. Watch the **Logs** in ESPHome — you should see:
   ```
   Connected to Blueriiot sensor
   raw_hex: 33.E0.06.19.08.25.0B.73.00.FF.0D.10
   temp: 17.60
   ph: 6.89
   orp: 717.5
   salt: 6.4
   bat: 44.4
   ```
3. The switch turns itself **off** after one successful reading
4. All five sensor entities now have values

If nothing happens within ~30 seconds: **briefly dunk the Blueriiot probe in water** or short-circuit the gold contacts at its bottom. The sensor only advertises when it's "active", and connection requires it to be receiving advertisements.

### 8. Automate periodic readings

Copy [`homeassistant/automation.yaml`](homeassistant/automation.yaml) into **Settings → Automations → ⋮ → Edit in YAML**. Default is every 30 minutes — adjust the trigger if you want different intervals.

> **Why every 30 min?** Blue Connect's coin cell lasts ~12 months at this rate. Polling every minute kills the battery in weeks.

### 9. (Optional) Install the dashboard card

1. Install **HACS** if you haven't, then add `card-mod` and `multiple-entity-row` from HACS Frontend.
2. Add the helper toggle: **Settings → Devices & Services → Helpers → + Create Helper → Toggle** with name `blueriiot_tipps`.
3. Open any dashboard → **Edit → Add Card → Manual** → paste the contents of [`homeassistant/dashboard-card.yaml`](homeassistant/dashboard-card.yaml).

The card shows live values, color-coded ranges, dosing recommendations, and toggleable help notes.

---

## Calibration

The salinity formula uses divisor `/18.0`, calibrated to match the official Blueriiot app on a Blue Connect Plus Salt. If your reading is off, here's how to recalibrate:

1. Take a reading with the **official Blueriiot app** — note the salinity value (e.g. `6.4 g/L`)
2. At the same time, check the ESPHome **Logs** for the `raw_hex` line — find the pair of bytes 7-8
3. Decode the raw value: `raw = (byte8 << 8) + byte7`
4. New divisor: `raw / app_value`

**Example:**
```
raw_hex: 33.E0.06.19.08.25.0B.73.00.FF.0D.10
                            ^^ ^^
                          byte7 byte8
raw = (0x00 << 8) + 0x73 = 115
app shows: 6.4 g/L
divisor = 115 / 6.4 = 17.97
```

Then update the lambda in `blueconnect.yaml`:
```cpp
float salt = (float)((int16_t)(x[8]<<8) + x[7]) / 17.97;
```

> **Note**: Sensor readings have natural ±0.1 g/L jitter. Don't chase decimals — pool salinity changes over weeks, not minutes.

---

## How the BLE protocol works

Each notification is **12 bytes**, little-endian, pushed by the sensor when we write `0x01` to the trigger characteristic:

| Byte(s) | Meaning | Formula | Example (`33.E0.06.19.08.25.0B.73.00.FF.0D.10`) |
|---|---|---|---|
| 0 | Header / status | – | `0x33` |
| 1-2 | Temperature | `raw / 100` | `0x06E0 = 1760 → 17.60 °C` |
| 3-4 | pH (raw) | `(2048 - raw) / 232 + 7` | `0x0819 = 2073 → 6.89` |
| 5-6 | ORP (raw) | `raw / 3.86 - 21.58` | `0x0B25 = 2853 → 717.5 mV` |
| 7-8 | Salinity (raw) | `raw / 18.0` (calibrated) | `0x0073 = 115 → 6.4 g/L` |
| 9-10 | Unknown / reserved | – | `0x0DFF` |
| 11 | Battery | `raw / 36 * 100` | `0x10 = 16 → 44.4 %` |

BLE service/characteristic UUIDs:

- **Service**: `F3300001-F0A2-9B06-0C59-1BC4763B5C00`
- **Trigger** (we write `0x01`): `F3300002-F0A2-9B06-0C59-1BC4763B5C00`
- **Notify** (we read): `F3300003-F0A2-9B06-0C59-1BC4763B5C00`

---

## Troubleshooting

### "Secret 'blueriiot_mac' not defined"

Your `secrets.yaml` is missing entries. See step 2.

### "Error resolving IP address of blueconnect.local"

mDNS isn't propagating, or the ESP isn't on WiFi yet. Check via your router's DHCP table for the assigned IP, then either:

- Use the IP directly in ESPHome's *Install → Manual IP*
- Or add a static IP under `wifi:` in the YAML:
  ```yaml
  wifi:
    manual_ip:
      static_ip: 192.168.X.XXX
      gateway: 192.168.X.1
      subnet: 255.255.255.0
  ```

### "Missing toolchain directory 'None'" during compile

PlatformIO's toolchain cache is corrupt. Delete it and retry:

```bash
# Inside the ESPHome add-on container:
rm -rf /data/cache/platformio /root/.platformio/tools/toolchain-xtensa-esp-elf
```

Then restart the ESPHome add-on and compile again.

### Sensor connects but readings stay empty

The Blueriiot probe must be **active** (in water or contacts shorted). Dry-stored sensors only respond after physical wake-up. Once a reading completes, the next ones work fine even out of water for ~5 minutes.

### Salinity value is off by a factor

See **Calibration** above — re-derive the divisor from a fresh raw_hex log.

### HA shows fewer decimals than ESPHome

HA caches the entity's display precision from the first registration. Fix:

**Settings → Devices & Services → Blueconnect → [sensor] → ⚙ → Display precision**

---

## File overview

```
.
├── esphome/
│   ├── blueconnect-scan.yaml      # Step 3: discover MAC
│   ├── blueconnect.yaml           # Step 5: production firmware
│   └── secrets.yaml.example       # Template for /config/esphome/secrets.yaml
├── homeassistant/
│   ├── automation.yaml            # Step 8: 30-minute polling
│   ├── input_boolean.yaml         # Helper toggle for dashboard
│   └── dashboard-card.yaml        # Step 9: full Lovelace card
├── LICENSE
└── README.md
```

---

## Credits & related work

- BLE protocol reverse-engineering: original work by various community members on the [ESPHome forum](https://community.home-assistant.io/) and the [Blueriiot Python project](https://github.com/wjtje/blueriiot-ble).
- This implementation uses ESPHome's native `ble_client` and `text_sensor` notify pattern.

---

## License

MIT — see [LICENSE](LICENSE).
