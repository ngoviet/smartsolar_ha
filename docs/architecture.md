# smartsolar_ha — Architecture & Data Flow

> Tách khỏi `CLAUDE.md` ngày 2026-10-13 (P2.1 chuẩn hóa D:\code) — giữ nguyên nội dung. Mục lục: [../CLAUDE.md](../CLAUDE.md)

## Architecture & Data Flow

### API (`api.smartsolar.io.vn`)

1. **Auth**: `POST /Auth/Login?Key=Content-Type` with `{username, password}` → token + expiry
2. **Token refresh**: Auto-refresh when within 7 days of expiry
3. **Retry logic**: Exponential backoff for network errors and 5xx responses. With `RETRY_MAX_ATTEMPTS = 3` the waits are 1 s then 2 s (`RETRY_BACKOFF_FACTOR ** attempt` over `attempts - 1` gaps)
4. **Device mode**: `GET /Device/Status?deviceGuid={guid}` → `{lastMessage: {dataStreams: [...]}, mqttConnection: {username, password}}`
5. **Project mode (by ID)**: `GET /Metric/ProjectMetrics?projectId={id}` → `{synthesisStreams, deviceLogs}`
6. **Project mode (by devices)**: `GET /Metric/SynthesisMetrics?deviceType={type}&deviceGuids={id1}&deviceGuids={id2}` → same structure
7. **MQTT Device Status**: `GET /Device/Status?deviceGuid={guid}` (used in project mode to extract mqttConnection credentials)

### MQTT (`mqttx.smartsolar.io.vn:8084`)

1. **Transport**: WebSocket Secure (WSS) at path `/mqtt`
2. **Credentials**: Auto-discovered from REST API `mqttConnection` field — username `web_app`, password base64-encoded
3. **Topics**: `manhquan/device/mppt_charger/log/+/<deviceGuid>` (single-level `+` wildcard for model)
4. **Payload format A** (newer firmware, `command: update_device_metrics`): `{dataStreams: [{name, value}, ...], signalQuality, command, deviceGuid, espId, firmwareVersion, messagesCounter}` — `signalQuality` is at the top level, NOT inside `dataStreams`
5. **Payload format B** (older firmware, `command: updateDeviceLog`): `{dataStreams: [{stream, name, value, unit}, ...], deviceGuid}` — carries **no** `signalQuality`, so that device's WiFi sensor stays `unknown`
6. **Payload format C** (legacy): flat dict with keys like `charging_power`, `yield_today`, mapped through `MQTT_FIELD_MAPPING`
7. **Restart-less upgrades are not supported: the MQTT subscription set and the
   entity set are both fixed at setup time.** A device that appears in a later
   poll is merged into `coordinator.data` and accepted by
   `coordinator._is_tracked_device()`, but it gets **no entities** (sensor.py
   only adds entities during `async_setup_entry`) and is **not subscribed to**
   for live data until Home Assistant restarts. Deliberate: per-device sensors
   are named `PV1`, `PV2`, … from the sorted GUID order, so inserting a device
   with a lower GUID would renumber the existing entities and break every
   dashboard reference to them.
8. **Reconnection**: Auto-reconnect every 5s on disconnect; graceful degradation to REST polling

> ⚠️ **The broker is shared by every SmartSolar customer.** A subscription to
> `manhquan/device/mppt_charger/log/+/#` receives hundreds of foreign devices.
> Only the per-device topics for this config entry are subscribed, and
> `coordinator._is_tracked_device()` rejects any other GUID in the callback.

### Data Flow

```
Config Entry (username, password, mode, chipset_ids/project_id)
    ↓
    ├── SmartSolarAPI ──────────────────────────────────────────────┐
    │   (login → token → periodic refresh, retry w/ backoff)        │
    │   POST /Auth/Login, GET /Metric/*, GET /Device/Status         │
    ↓                                                                │
    ├── SmartSolarMQTTClient ───────────────────────────────────────┤
    │   (WSS connect → subscribe topics → parse payloads)           │
    │   mqttx.smartsolar.io.vn:8084/mqtt                           │
    │   Auto-discovers mqttConnection from API response             │
    ↓                                                                │
SmartSolarDataUpdateCoordinator                                     │
    (polling every N seconds + real-time MQTT merge)                │
    async_process_mqtt_data() → _merge_mqtt_into_data()             │
    _schedule_mqtt_notify() → async_update_listeners()  (1 Hz cap)  │
    ↓
Sensor Entities (CoordinatorEntity + RestoreEntity, ×13 types)
Number Entity (Update Frequency, 1-30s)
```

### Sensor Types

13 sensor types per device: 10 live metrics (`pv_voltage`, `pv_current`, `bat_voltage`, `bat_current`, `charge_power`, `today_kwh`, `total_kwh`, `temperature`, `signal_quality` (WiFi %, via MQTT or REST), `status`) plus 3 daily stats (`peak_power_today`, `avg_power_today`, `production_hours_today`)

### Four Sensor Classes

| Class | Mode | Data Source |
|-------|------|-------------|
| `SmartSolarDeviceSensor` | Device | `data.lastMessage.dataStreams` |
| `SmartSolarProjectSynthesisSensor` | Project | `data.synthesisStreams` (fallback: per-type aggregation from deviceLogs) |
| `SmartSolarProjectDeviceSensor` | Project | `data.deviceLogs[deviceGuid].dataStreams` |
| `SmartSolarStatsSensor` | Both | `coordinator.get_daily_stats(guid)` — peak/avg/production-hours for today |

#### Cross-device aggregation (`const.AGGREGATION`)

When the project total comes from the individual device logs, each sensor type
has its own strategy. **Voltage and temperature are averaged, never summed** —
both chargers sit on the same 24 V bus, so summing reports ~53 V:

| Strategy | Sensor types |
|----------|--------------|
| `sum` | pv_current, bat_current, charge_power, today_kwh, total_kwh |
| `average` | pv_voltage, bat_voltage, temperature, signal_quality |
| `max` | status (worst case) |

Sensors in `const.UNRELIABLE_SYNTHESIS_SENSORS` (charge_power, currents,
signal_quality) always aggregate locally because the server's
`synthesisStreams` value is frequently stale or absent.
