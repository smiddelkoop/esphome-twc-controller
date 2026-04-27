# Changes vs upstream (jnicolson/esphome-twc-controller @ 724b8d7)

This fork was created from commit [`724b8d7`](https://github.com/jnicolson/esphome-twc-controller/commit/724b8d7) of the upstream repository, which has not been actively maintained since 2023. The changes below were identified and validated on an ESP32-C6 running ESPHome 2026.3.3, integrated with Home Assistant via `aioesphomeapi`.

---

## 1. Secondary presence re-handshake on TWC reset

**File:** `components/twc-controller/twc_protocol.cpp`

**Problem:** When the TWC resets itself mid-session (e.g. after a transient UART checksum error or a power glitch), it re-announces its presence by sending a `SECONDARY_PRESENCE` packet. The upstream code recognised this as a known charger and logged a warning, but took no further action — it simply continued sending heartbeats. However, the TWC protocol requires the primary controller to first respond with `PRIMARY_PRESENCE` packets (`SendPresence()` + `SendPresence2()`) before it will accept heartbeats again. Without this handshake, the TWC silently ignores all subsequent heartbeats, charging stops, and the session cannot recover without a full ESP32 reboot.

**Fix:** When a known charger re-announces its presence, `SendPresence()` and `SendPresence2()` are now called immediately, re-establishing the primary/secondary relationship. Charging resumes automatically within one heartbeat cycle.

```cpp
// Before (upstream):
} else {
    ESP_LOGW(TAG, "Known charger re-announced presence (TWC reset): ID: %04x, available_current=%dA",
        presence->twcid, available_current_);
}

// After (this fork):
} else {
    ESP_LOGW(TAG, "Known charger re-announced presence (TWC reset): ID: %04x, available_current=%dA - sending presence re-handshake",
        presence->twcid, available_current_);
    SendPresence();
    SendPresence2();
}
```

---

## 2. Serial text sensor: raw bytes replaced with hex string

**File:** `components/twc-controller/twc_protocol.cpp` / YAML configuration

**Problem:** The `serial` text sensor published the raw binary bytes of the TWC serial number directly as a string (e.g. `p\x87\x88\xd3Z\x93G\x8e#`). When ESPHome 2026.x serialises this value and sends it to Home Assistant, `aioesphomeapi` rejects the malformed UTF-8 payload and closes the connection. This produces a rapid connect/disconnect loop (~150 ms cycles) with `CONNECTION_CLOSED errno=128` in the HA logs, making all sensors unavailable.

**Fix:** A `lambda` filter is added to the `serial` sensor in the YAML configuration, converting each raw byte to its two-character uppercase hex representation:

```yaml
twc-controller:
  serial:
    name: "Serial"
    filters:
      - lambda: |-
          std::string out;
          for (char c : x) out += str_sprintf("%02X", (unsigned char)c);
          return out;
```

**Note:** The `serial` sensor must remain present in the YAML even if its output is not needed. The component unconditionally calls `publish_state()` on the sensor object regardless of YAML configuration; removing it causes a null pointer dereference and a hard fault (`Load access fault` in `DecodeSerialNumber()` at `twc_protocol.cpp:383`).

---

## 3. `max_allowable_current` sensor removed from YAML

**File:** YAML configuration

**Problem:** The `max_allowable_current` sensor was referenced in YAML examples but the component's C++ code calls `publish_state()` on it unconditionally. Unlike `serial`, the upstream example YAML does not consistently define it, meaning any configuration that omits it while the component references it will crash with a null pointer dereference on the next sensor update.

**Fix:** The sensor is removed from the YAML. If you need to expose the maximum allowable current as a Home Assistant entity, it must be explicitly defined in the YAML — silently omitting it is not safe with the current component implementation.

---

## 4. External component pinned to a specific commit hash

**File:** YAML configuration

**Problem:** Referencing an external ESPHome component via `ref: main` means every new ESPHome build pulls the latest HEAD of that branch. For an unmaintained repository, this provides no stability guarantee — a future incompatible change or ESPHome API update could silently break the component at the next build.

**Fix:** The component reference is pinned to commit `724b8d7`, the last known-good HEAD of the upstream repository at the time this fork was created:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/smiddelkoop/esphome-twc-controller
      ref: 724b8d7
    components: [twc-controller]
```

Use this fork's URL to incorporate all fixes above.

---

## Compatibility

Tested on:
- **Hardware:** ESP32-C6 (`esp32-c6-devkitc-1`)
- **ESPHome:** 2026.3.3
- **Home Assistant / aioesphomeapi:** current as of April 2026
- **TWC hardware:** Tesla Wall Connector Gen 2
