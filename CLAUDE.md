# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

ESP-IDF firmware (C) for the Franzininho WiFi LAB01 board (ESP32-S2). It reads a DHT11 sensor periodically, publishes temperature/humidity over MQTT, and publishes a retained alarm message when limits are exceeded. A subscriber (a second board, the same board, or `tools/alarm_subscriber.py`) drives an alarm LED from that topic. README and code comments/logs are in Portuguese. The C and Kconfig files are ASCII-only (no accents); the README uses accents.

## Commands

Requires ESP-IDF 5.x or 6.x with the environment exported (`IDF_PATH` set).

```bash
idf.py set-target esp32s2          # once; sdkconfig.defaults already sets esp32s2
idf.py menuconfig                  # "Monitor Ambiental MQTT": Wi-Fi, broker, limits, board role
idf.py build
idf.py -p COMx flash monitor
idf.py fullclean                   # after changing Kconfig choices or switching IDF versions
```

- On IDF 6.0+, `mqtt` and `cJSON` come from the Component Registry (`main/idf_component.yml`), so the first build needs internet access. They are downloaded into `managed_components/` (gitignored).
- There are no unit tests or lint config. To check a change, build it, then run it on hardware or with the PC subscriber:
  `pip install paho-mqtt && python tools/alarm_subscriber.py --broker broker.hivemq.com --base franzininho/lab01`
- `sdkconfig` is gitignored. Put persistent defaults in `sdkconfig.defaults`, not in `sdkconfig`. The console uses native USB CDC (`CONFIG_ESP_CONSOLE_USB_CDC=y`).

## Architecture

- **Board role is selected at compile time** in `main/Kconfig.projbuild` (`APP_ROLE_*` choice). Two derived hidden symbols control the code: `CONFIG_APP_HAS_SENSOR` (publisher: `sensor_task`, alarm logic, the "Sensor e limites" menu) and `CONFIG_APP_HAS_ALARM_LED` (subscriber: LED GPIO and subscription to `<base>/alarme`). `main/main.c` is split into `#if` blocks on these two symbols. Any new code that is specific to one role must be guarded the same way, or builds for the other roles will fail. Config symbols such as `CONFIG_TEMP_LIMIT_C` exist only when their menu's `depends on` is satisfied.
- **Startup order** (`app_main`): NVS init, then LED init, then `wifi_sta_connect()` (blocks until an IP is assigned and auto-reconnects on disconnect), then `mqtt_start()`, then the sensor task. MQTT connection state is tracked in an event group bit. The sensor task publishes only while that bit is set; otherwise it skips the publish and does not queue it.
- **The MQTT topic/payload contract is shared by three consumers**: the publisher (`publish_alarm` in `main.c`), the C subscriber (`handle_alarm_message`), and `tools/alarm_subscriber.py`. Alarm payload is JSON `{"estado":"ON"|"OFF", "motivo", ...}`, QoS 1, retained. Both subscribers also accept plain `ON`/`OFF`/`1`/`0` for manual testing. `<base>/status` carries `online`/`offline` via the Last Will. If you change topics or payload fields, update all three consumers and the README table.
- **Alarm state machine**: the alarm is published only when its state changes (plus once at the first reading). It uses hysteresis: it turns on when a value is `> limit` and turns off only when the value drops to `<= limit - CONFIG_ALARM_HYSTERESIS`. If a publish fails, `alarm_pending` stays set and the alarm is published again on the next reading.
- **MQTT client ID** is derived from the STA MAC (`lab01-xxxxxx`) so that two boards can share a broker.
- **DHT11 driver** (`main/dht11.c`) bit-bangs the protocol on an open-drain pin with pull-up. The ~5 ms read runs inside `portENTER_CRITICAL`, so code in that section must not block, log, or call FreeRTOS APIs. A bit is decoded as 1 when its high pulse lasts more than 40 µs. The caller (`sensor_task`) retries up to 3 times, 2 s apart.
- `main/CMakeLists.txt` selects component names by IDF major version (`espressif__mqtt espressif__cjson` on 6+, `mqtt json` on 5.x). Keep that conditional when you add dependencies that moved to the registry.
