---
title: "Props that must never fail: our ESP32 firmware base"
date: 2026-11-12
lang: en
tags: [esp32, esp-idf, firmware, c, freertos, mqtt, ota, electronics, escape-room]
---

> **Alternative titles** (pick one, then delete this block)
>
> 1. Hardware first, network second: our ESP-IDF base for escape-room props
> 2. The board that answered ping: ESP32 firmware lessons from the field
> 3. Safe mode, signed OTA and polling on purpose: firmware for hostile rooms

# Props that must never fail: our ESP32 firmware base

In an escape room, the *props* are where software meets the physical world: a keypad, a chest with an electromagnetic lock, canopic jars that recognize the statuettes placed on them, a cannon, a kraken that blinks its eyes. At Team Break, most of these props are driven by an **ESP32** programmed in C with **ESP-IDF**. So that they all behave alike (same configuration, same web page, same behaviour when something breaks), they share a common **firmware base**.

In the [first article](./team-break-architecture.en.md) of this series I skimmed over the electronics. Here I go into the details: why this base exists, how it's built, and what the field has taught us.

## The environment: hostile, and nobody on site

An escape-room prop lives in conditions you don't often see in software:

- **Everything runs on 12 V**, nothing on USB. Power supplies are shared with electromagnets, motors and relays.
- **Cables are long** and run next to inductive loads that switch all the time. Any glitch is a potential phantom input.
- **The room is full of metal** (and of other props that emit): Wi-Fi isn't always welcome.
- **Power gets cut abruptly**, sometimes several times a day.
- **Players handle everything**, and not always gently.
- **Some rooms have no Internet**, and **nobody can press reset** in the middle of a session.

Add a variety of wiring that no documentation really captures: depending on the site, the same sensor is mounted upside down, or as a normally-closed contact. And a few mechanical surprises: at La Défense, the daggers of one puzzle are small car central-locking actuators, which must *never* be powered continuously.

On the hardware side, the standard is the **ESP32-WROOM** with 4 MB of flash. Where relays are needed we use **Kincony KC868-A16** boards: an ESP32, 16 relays and 16 inputs on PCF8574 I2C expanders, plus Ethernet through a LAN8720 PHY. A few Arduino Unos with Ethernet shields remain on older installations, along with some third-party I/O panels.

## Why a shared base

At first, every prop had its own firmware, copied from a template. Configuration was described with **X-macros**:

```c
#define TB_CONFIG_CHAR_ITEMS                                   \
    TB_CONFIG_CHAR_ITEM(char *, ssid, "")                      \
    TB_CONFIG_CHAR_ITEM(char *, hostname, "New")               \
    TB_CONFIG_CHAR_ITEM(char *, mqtt_broker, "mqtt://mqtt.eclipse.org")
#define TB_CONFIG_ITEMS                                        \
    TB_CONFIG_ITEM(int, mqtt_port, 1883)                       \
    TB_CONFIG_ITEM(int, ma_nouvelle_valeur, 42)
```

That produced a flat struct (`config.ssid`, `config.mqtt_port`) with no validation and no migration. To add a field to a prop, the documentation advised… editing the shared code. The network component had become a "god component" that bundled Wi-Fi, MQTT and the HTTP server, with 11 dependencies and a circular dependency on the Ethernet driver. Every copy drifted, and every fix had to be carried over by hand.

The current base grew out of that. It's a repository of ESP-IDF components that every prop firmware includes as a **Git submodule**. On the prop side, one line in the top-level `CMakeLists.txt` is enough:

```cmake
set(EXTRA_COMPONENT_DIRS ${CMAKE_SOURCE_DIR}/shared_components)
```

The prop only brings what is specific to it: its pinout, its game logic, its messages and its configuration section. Splitting the network component removed 1,052 lines and brought its dependencies down from 11 to 3.

The base builds with **ESP-IDF 5.5 and 6.0**. Version 6 moved several components out of the framework (cJSON, the MQTT client, the LAN87xx driver) into the component registry; they're declared conditionally in the `idf_component.yml` files (`if: "idf_version >=6.0"`). One rule is written into every spec: **updating the submodule must require no change to the host firmwares**.

Today it has about twenty components, roughly 13,000 lines of C and 6,900 lines of web UI:

| Area | Components |
|---|---|
| Network | `tb_network` (TCP/IP stack, mDNS, interfaces), `tb_wifi`, `tb_lan8720`, `tb_mqtt`, `tb_ntp`, `tb_http_server` |
| System | `tb_storage` (LittleFS), `tb_config`, `tb_log`, `tb_watchdog`, `tb_recovery`, `tb_infos` |
| Hardware | `tb_gpio`, `tb_i2c`, `tb_pcf8574`, `tb_sx1509`, `tb_neopixel`, `lib8tion`, `tb_rfid`, `tb_pn532` |

## Interfaces in C, with function pointers

So that a prop doesn't need to know whether its network runs over Wi-Fi or Ethernet, whether a pin is internal or on an I2C expander, or which RFID reader model it uses, the base defines old-school **interfaces**: structs of function pointers.

```c
// Network: Wi-Fi or Ethernet, same contract
struct tb_network_interface_t {
  get_state get_state;   // NOT_CONFIGURED / DISCONNECTED / CONNECTING / CONNECTED
  get_infos get_infos;   // cJSON* for the web page
  get_name  get_name;
  void *datas;           // implementation data
  void *netif;           // esp_netif_t*
};

// GPIO: internal pins, PCF8574, SX1509…
struct tb_gpio_interface_t {
  read_gpio read;              write_gpio write;
  get_gpio_type get_type;      set_gpio_type set_type;
  get_interface_name get_name; get_gpio_count get_count;
  set_interface_name set_name; void *datas;
};

// RFID: PN532 today, something else tomorrow
struct tb_rfid_interface_t {
  tb_rfid_read_uid read_uid;   tb_rfid_get_reader_state get_state;
  tb_rfid_reset_reader reset_reader; tb_rfid_get_reader_count get_count;
  tb_rfid_get_interface_name get_name;
  void *datas;
};
```

An I/O expander is just another implementation of `tb_gpio_interface_t`: the rest of the code (filters, callbacks, web page) treats it exactly like the ESP32's own pins.

## Boot order is a safety decision

The initialization order is fixed and documented, with a reason for each step:

1. **Storage** first: configuration, logs, recovery state and the web page are all files.
2. **Recovery** before configuration: it may replace a broken configuration with the last known-good one. Reading the configuration first would load the broken one.
3. **Configuration, logs, watchdog.**
4. **Hardware.**
5. **Network.**
6. **NTP and MQTT**, except in safe mode.
7. **The HTTP server** last, then the boot-success marker is armed.

Step four is the most important. The documentation explains it in one sentence:

> **Hardware before network.** An output should reach its safe state (locks closed, motors off) as early as possible, not after a Wi-Fi connection that can take several seconds or never come.

The Kincony board firmware pushes the idea even further: it initializes the relays *before storage*, and forces them to their rest state in the first few milliseconds.

```c
tb_gpio_init();
tb_i2c_init(&(tb_i2c_config_t){.sda = 4, .scl = 5});
in1  = init_pcf8574_with_retry(0x21, "in1");   in2  = init_pcf8574_with_retry(0x22, "in2");
out1 = init_pcf8574_with_retry(0x24, "out1");  out2 = init_pcf8574_with_retry(0x25, "out2");
configure_output_bank(out1, 0); configure_output_bank(out2, 8);   // write 1 = relays at rest
configure_input_bank(in1, 0);   configure_input_bank(in2, 8);
tb_storage_init_flash();
tb_recovery_init();  bool safe = tb_recovery_is_safe_mode_active();
tb_config_init(&config); tb_log_init(); tb_watchdog_init();
tb_network_init();
tb_network_add_interface(tb_lan8720_init(&eth_config));
tb_network_add_interface(tb_wifi_init());
if (!safe) { tb_ntp_init(); /* … then MQTT and its callbacks */ }
if (tb_http_server_init() == ESP_OK) tb_recovery_mark_boot_successful();
```

## Schema-driven configuration

Each component declares its configuration section with declarative macros. Here's the MQTT client's:

```c
_Static_assert(MQTT_MAX_TOPIC_SIZE - 1 == 255, "sub_topic max length must follow MQTT_MAX_TOPIC_SIZE");
TB_CONFIG_GENERATE_PARSER(tb_mqtt, TB_MQTT, "tb_mqtt",
    TB_FIELD_STRING(tb_mqtt_config_t, broker, "localhost", 256),
    TB_FIELD_UINT32_RANGE(tb_mqtt_config_t, port, 1883, 1, 65535),
    TB_FIELD_STRING_LEN(tb_mqtt_config_t, sub_topic, "/sub_topic", 1, 255),
    TB_FIELD_STRING(tb_mqtt_config_t, pub_topic, "/pub_topic", 256)
);
TB_CONFIG_DEFINE_GETTER(tb_mqtt, "tb_mqtt", false);
```

A prop declares its own section the same way, in its own code, without touching the base. For the kraken in *Le Trésor des pirates*, that gives a tunable window and cooldown:

```c
TB_CONFIG_GENERATE_PARSER(kraken, KRAKEN, "kraken",
    TB_FIELD_UINT32_RANGE(kraken_config_t, window,   5000, 500, 60000),
    TB_FIELD_UINT32_RANGE(kraken_config_t, cooldown, 3000, 100, 30000));
TB_CONFIG_DEFINE_GETTER(kraken, "kraken", true);
// usage: uint32_t window_ms = kraken_get_config()->window;
```

At build time, CMake discovers every `*_config_schema.h` file, and Python generators produce two things: a validation header for the firmware, and **JSON Schemas** from which the prop's web page builds its forms. Adding a setting means adding one line: parsing, validation, the typed getter and the form field all follow. The resulting configuration file looks like this (example values):

```json
{
  "tb_mqtt":        { "broker": "mqtt://<broker>", "port": 1883,
                      "sub_topic": "/my_prop_sub", "pub_topic": "/my_prop_pub" },
  "tb_network":     { "hostname": "room-my-prop" },
  "tb_ntp":         { "enabled": true, "timezone": "CET-1CEST-2,M3.5.0/02:00:00,M10.5.0/03:00:00" },
  "log":            { "enabled": true, "max_file_size": 65536, "log_level": "warn" },
  "tb_recovery":    { "stable_uptime_s": 3600 },
  "my_prop":        { "tolerance_ms": 800 }
}
```

### Always have a plan B, and a plan C

A configuration can get lost in many ways: a write interrupted by a power cut, a truncated file uploaded from the web page, a typo. So loading follows a fallback chain, decided by a pure function:

1. the current file, if it's valid;
2. otherwise the **last known-good configuration** (`tb_config.last-known-good.json`);
3. otherwise the **factory defaults** (`tb_config.factory-defaults.json`);
4. otherwise the schema defaults, with the fallback Wi-Fi access point reachable.

The invalid file isn't deleted: it's renamed `tb_config.invalid.json` for diagnosis. And a file that exists but *can't be read* (out of memory, I/O error) is never overwritten, because it may be perfectly valid.

Changes from the web page are merged into the current configuration, validated, then written atomically (temporary file, then rename). The "Apply now" button re-applies Wi-Fi, MQTT, network, NTP and logging live; if the apply fails, the configuration rolls back to the last valid version.

## Never stay stuck

### Watchdog and core dump

The ESP-IDF task watchdog is set to 10 s with panic, and watches both cores' idle tasks as well as the base's own tasks (GPIO, MQTT, live logs). On a crash, a core dump is written to a dedicated 64 KB partition, readable afterwards with `idf.py coredump-info`.

### Safe mode

A watchdog that reboots the board is good. A board that reboots in a loop because its configuration makes it crash is a prop lost for the day. So the `tb_recovery` component counts failed boots:

- counted as failures: panic, watchdogs, brownout, and an **unannounced** software restart;
- not counted: power-on, external reset, and announced restarts (reboot from the web page, OTA, factory reset);
- a boot only counts as successful after `stable_uptime_s`, **60 minutes by default**. That's the length of a game: a crash in the middle of a session counts as a failure.

On the **second consecutive failure**, the board boots into safe mode: configuration restored, Wi-Fi and web page up, but **no MQTT**. It stays reachable for diagnosis and, above all, out of the game: an unstable prop must not send phantom events to the room engine.

The decision is a pure function, with no hardware call at all:

```c
// A software restart announced by tb_recovery_note_intentional_restart()
// (reboot, OTA, factory reset from the web page) is not a crash: like a
// power-on, it resets the counter.
const bool announced_restart =
    previous->intentional_restart_pending && reason == ESP_RST_SW;
decision.user_requested = previous->safe_mode_requested;

if (previous->boot_marked_successful || announced_restart ||
    tb_recovery_reason_resets_counter(reason)) {
  next->failed_boot_count = 0;
} else if (!decision.user_requested && tb_recovery_reason_is_crash(reason)) {
  next->failed_boot_count = previous->failed_boot_count + 1;
  decision.counted_as_failure = true;
}
decision.repeated_failures = next->failed_boot_count >= TB_RECOVERY_SAFE_MODE_THRESHOLD;
decision.safe_mode_active  = decision.user_requested || decision.repeated_failures;
```

Because it's pure, it can be tested on a PC, case by case, as we'll see below.

### Logs that survive a reboot

The `TB_LOGx` macros are used like `ESP_LOGx`, but also push each line into an 8 KB RAM buffer. A low-priority writer task flushes that buffer to `app.log` every 500 ms, or as soon as it's a quarter full, with rotation at a configurable size. Just before a deliberate restart, the buffer is flushed within a second.

This asynchronous design comes from a bug found on the bench: the old version wrote to flash under a mutex, and on a board without network **65 log lines were lost in 30 minutes** to contention. Today the caller never waits more than 10 ms, and lost lines are *counted*: an "N log line(s) lost" line is written, and a counter is exposed by the API. You can lose logs; you mustn't lose the fact that you lost some.

## Networking: Ethernet first, Wi-Fi when needed

### Ethernet

In a room full of metal, Ethernet is the default whenever the board supports it. The `tb_lan8720` component drives the ESP32's internal MAC with a LAN8720 PHY (Kincony, Olimex ESP32-POE, WT32-ETH01). ESP-IDF v6 removed the Kconfig options for the RMII clock; the component re-declares them under the same names, loaded only on v6, so existing firmwares' `sdkconfig.defaults` keep working unchanged.

### Wi-Fi, and a bit shift

Wi-Fi reconnects forever with a backoff of 1, 2, 4, 8, 16 s, then every 30 s. The first version computed the delay as `BASE * (1 << retry_num)`, with no bound. After about 13 minutes without an access point the shift overflowed, the delay dropped to zero, and the board retried in a tight loop. The logic is now a pure, bounded function:

```c
uint32_t tb_wifi_retry_delay_ms(int retry_num) {
  if (retry_num < 0) {
    retry_num = 0;
  }
  // 1000 << 5 = 32000 is already above the maximum: no shift beyond 4
  if (retry_num >= 5) {
    return TB_WIFI_RETRY_MAX_DELAY_MS;
  }
  uint32_t delay_ms = (uint32_t)TB_WIFI_RETRY_BASE_DELAY_MS << retry_num;
  return (delay_ms > TB_WIFI_RETRY_MAX_DELAY_MS) ? TB_WIFI_RETRY_MAX_DELAY_MS : delay_ms;
}
```

### MQTT, deliberately simple

The prop's MQTT client does few things, but does them predictably: QoS 0, exact topics (no wildcards), at most 10 callbacks, messages up to 1 KB. A task watched by the watchdog checks the connection every 5 s and resubscribes on every reconnect.

A few design decisions:

- **A new board has no broker.** Initialization succeeds anyway, and the client starts as soon as a broker is saved from the web page, with no reboot.
- **The subscription topic can change live.** The client unsubscribes from the old one and moves every callback to the new one.
- **Callbacks run in the MQTT task.** That's pitfall number one, documented in bold: while a callback runs, no other message is read. Store the command there (a variable, a FreeRTOS queue) and let one of the prop's own tasks act on it.

The code review found two tasty bugs on the receive path. The payload buffer wasn't null-terminated, so `"30"` could be read as `"30_lights_on…"`, the leftover of the previous message. And topic comparison didn't check lengths: a topic that was a prefix of another one captured it.

### NTP, locally

Logs need real time. The base uses SNTP with a POSIX timezone string; before sync, lines are stamped `boot+<ms>ms`. Rooms without Internet need a local NTP server, and to measure durations you use `esp_timer_get_time()`, never wall-clock time.

## Inputs and outputs: polling, on purpose

### Why not interrupts for everything?

The `tb_gpio` component reads every declared input from a task that runs every 10 ms, then applies a filter: debounce (50 ms by default, configurable per pin) or an exponential moving average for analog inputs. The question "why not interrupts?" comes up regularly. The answer is written in the roadmap, and it's entirely dictated by the field:

> Escape-room wiring is long and sits next to switching relays and electromagnets. Polling plus debounce ignores glitches; an interrupt fires on each one, and a floating or noisy line can flood the CPU until the interrupt watchdog resets the board.

On top of that, mechanical contacts bounce (you'd need a timed debounce behind the interrupt anyway), expanders and analog inputs would stay polled regardless, and props in production were validated that way. Interrupts do exist, but as an option, for pulses too short for polling: a ball crossing an infrared beam in about 15 ms. In that case the ISR pushes timestamped edges into a queue and keeps a per-pin edge counter.

For the wiring diversity mentioned earlier, an `invert_logic` option really inverts a pin's reads, callbacks and writes. A sensor mounted upside down at one site becomes a line of configuration, not a code branch.

### The relay that clicked

Kincony boards drive their relays through PCF8574s, 8-bit I2C expanders. The PCF8574 has no direction register: you write the whole port byte every time. The on-board review found two defects that were annoying in a room:

- **A relay clicked on every write.** The write function switched the pin back to output, which sent the bit as 0, *then* the real value.
- **Every relay pulsed at boot.** The local copy of the port started at `0x00`: the first write briefly pulled every output low.

The fix is short: a local *shadow* copy initialized to `0xFF`, and **one transmission per write**, changing only the targeted bit.

```c
esp_err_t tb_pcf8574_port_write(uint8_t *port, uint8_t io, int32_t value,
                                tb_pcf8574_tx_t tx, void *ctx) {
  if (port == NULL || tx == NULL || io >= PCF8574_LOGIC_NB_PINS) {
    return ESP_ERR_INVALID_ARG;
  }
  uint8_t next;
  if (value) {
    next = *port | (uint8_t)(1U << io);
  } else {
    next = *port & (uint8_t)~(1U << io);
  }
  esp_err_t err = tx(ctx, next);
  if (err == ESP_OK) {
    *port = next;
  }
  return err;
}
```

The shadow copy is only updated after a successful transmission. This logic is tested exhaustively: 256 port states, 8 pins and 2 values.

### The unplugged expander that rebooted the board

Another find: unplugging an I2C expander rebooted the board. I2C transactions used a timeout of `-1` (wait forever) inside a task watched by the 10 s watchdog. The component documentation now says it bluntly: "A `-1` timeout blocks forever… that reboots the board." Timeouts are now 50 ms, a failed read returns `-1` (unknown value), and an expander that stops answering produces *one* log line when it goes and one when it comes back, not one per tick.

### RFID: a silent reader is not a missing object

The RFID component handles PN532 readers (up to 8 on one SPI bus), with an automatic reset after 20 failed reads that re-applies the gain and RF level. Two choices are worth pointing out:

- the PN532's passive retries are limited to 1, because the default behaviour (retry forever) "makes an empty reader look like a dead one";
- in the API, **"no tag" is not an error**: `ESP_OK` with a zero-length UID. An error means *the reader* isn't answering. That's what lets a prop report a broken reader instead of waiting forever for an object.

Expected UIDs are stored as strings in the configuration. The canopic jars in our Egyptian room (four RFID readers) work this way: replacing a lost statuette is a configuration change, not a new firmware.

### And light

The NeoPixel component drives WS2812 strips in RGB, hex or HSV, with FastLED's 8-bit math library ported to C for animations. Brightness is capped at 50/255 by default. That's not just aesthetics: at 60 mA per LED in full white, a strip switching on all at once can make a power supply shared with an electromagnet sag.

## A web page in every prop

Every prop serves a web page from its LittleFS partition, reachable by its mDNS name (`http://<hostname>.local/`):

- **live state** of the I/O over SSE (at most 10 clients per stream, a keepalive every 25 s);
- the **configuration editor**, with forms generated from the schemas;
- **logs**, a file browser, reboot, safe mode, factory reset;
- **OTA update** of the firmware, and a separate update of the web UI (`www_update.tar`).

It's the first-line diagnostic tool: on the room network (or over the VPN), a browser is enough to see what the prop sees.

## Signed OTA and reproducible releases

The partition table is the same for every prop: two 1.25 MB app slots (`ota_0`, `ota_1`), 1.4 MB of LittleFS for configuration, logs and the web page, and 64 KB for the core dump.

CI builds the firmware on every tag, with a pinned ESP-IDF, and publishes the full set of artifacts: application, bootloader, partition table, storage image, web UI archive and a complete flash package. It also produces a **manifest** (version, commit, size and SHA-256 of each artifact), **signed with RSA-3072**. The public key is injected at build time into a generated header, and the prop checks the signature with mbedTLS, then the artifact's size and hash, before accepting an update. Builds are reproducible (`CONFIG_APP_REPRODUCIBLE_BUILD=y`, no build date in the bootloader).

In the field, updates go through the web page: the firmware is written to the inactive slot, and the configuration is kept. One rule from our maintenance knowledge base is worth quoting: **never reflash the canopic jars over serial**. `idf.py flash` also rewrites the storage partition, and so erases the learned statuettes. Use OTA, or `idf.py app-flash`, which only touches the application.

What's still missing: automatic rollback to the previous slot if the new firmware fails to boot. It's on the roadmap.

## Plugging a prop into Maestro

On the room engine side, an ESP32 prop is just one more **electronics provider**, with its message format (`tb_esp32`) and its topics:

```json
{
  "key": "esp32_digicode",
  "message_format": "tb_esp32",
  "input_topics": [ "/games/chocolaterie/rooms/iceroom/digicode/state" ],
  "output_topic": "/games/chocolaterie/rooms/iceroom/digicode"
}
```

When the keypad publishes `{"module":"in2","pin":0,"value":"off"}`, Maestro's electronic agent derives the key `esp|min2|p0|voff`, matches it against the proxies declared in the gameflow, and publishes a business notification (`digicode_unlocked`) to the room orchestrator. In the other direction, the `trappe_unlock` command becomes `{"command":"set","data":[{"module":"out1","pin":0,"value":"on"}]}`. The same room happily combines third-party I/O panels and ESP32 props: each provider has its own broker, topics and format.

The Kincony board protocol is deliberately minimal:

```json
{"command":"set","data":[{"module":"out1","pin":2,"value":"on"},{"module":"out2","pin":3,"value":"off"}]}
{"command":"get","data":[{"module":"in1","pin":3}]}
{"command":"get_all_infos"}
```

`get_all_infos` returns the board's full state: platform (SDK version, free memory, uptime), network, MQTT, storage, and each GPIO with its filter, type and value. On the Maestro side, the technical page of the GM console uses `get` to read pin states back on demand.

## Testing firmware without a board

A good part of the base's logic is tested **without an ESP32**. ESP-IDF can build for a `linux` target, and we take advantage of it: 9 test apps, **124 Unity tests**, built and run on a PC by CI.

| Component | Tests |
|---|---|
| `tb_config` | 49 |
| `tb_gpio` | 21 |
| `tb_recovery` | 13 |
| `tb_mqtt` | 12 |
| `tb_pcf8574` | 8 (including one exhaustive test) |
| `tb_neopixel` | 7 |
| `tb_log` | 6 |
| `tb_http_server` | 5 |
| `tb_wifi` | 3 |

The method is always the same: rather than stubbing a whole component, **isolate the decision logic in a pure file** (`tb_wifi_logic.c`, `tb_recovery_decision.c`, `tb_pcf8574_logic.c`, `tb_config_source.c`) and test it directly. Test names cite the acceptance criteria of the specs (`test_crit10_safe_mode_at_two_failures`, `test_crit37_retry_delay_never_zero`).

CI also builds a firmware that uses *every* component, with ESP-IDF 5.5.2 **and** 6.0.2, checks schema discovery (including adding and removing a schema between two builds), and watches the flash budget: the Kincony board firmware must keep at least a 3.5% margin.

## A review on real hardware, and a priority rule

In October we put the base through a full review: documentation, code reading and bench tests on a development board. The outcome was a list of defects, some of which are described above. Two more are worth telling:

- **A truncated configuration took the board off the network.** Uploaded from the web page, it made every component fall back to hard-coded values. Worse: a save right after overwrote the last known-good configuration with `{}`. Hence the fallback chain described above, and the rule of never writing the last known-good configuration from a configuration that wasn't loaded.
- **"MQTT reconnected successfully" was logged too early**, when the reconnection request was accepted rather than when the connection was established. A log that lies is worse than no log.

Fixes were grouped into batches, with an ordering rule that sums up our trade pretty well:

> Ordering rule: what stops a prop in the middle of a session comes first, then what fails silently on props already in production (MQTT, network), then what breaks a build or a new project, then hardware drivers, then comfort and diagnostics.

The roadmap has 45 items: 31 done, 2 in progress, 12 to do (including OTA rollback and integration tests).

## For comparison: the Arduino that answered ping

Not all our props are on this base. The Chocolaterie elevator's hatches run on an Arduino Uno with a W5500 Ethernet shield, and they taught me a lesson I tell in the [first article](./team-break-architecture.en.md): the board froze now and then, yet still answered ping, because the W5500 handles ICMP on its own. The Ethernet library waited on the controller in loops with no timeout, and noise from an electromagnet was enough to hang it.

The fix looks, in miniature, like what the ESP32 base does: a 4 s hardware watchdog, short network timeouts, no more blocking loops, the electromagnet energized at the very start of `setup()`. And a breadcrumb that survives the reset:

```c
struct Persist { uint16_t magic; uint16_t bootCount; uint8_t step; };
Persist persist __attribute__((section(".noinit")));  // survives a reset, not a power-off
// setup(): MCUSR = 0; wdt_disable(); warmBoot = (persist.magic == 0xC0C1);
// loop():  wdt_reset(); persist.step = STEP_MQTT_LOOP; client.loop(); ...
```

On every connection, the board publishes a retained diagnostics message with the boot count, the type of the last reset and the step the code was at just before it (`reset_step`). All of it fits in 829 bytes of RAM out of the 2,048 available, thanks to `F()` strings and dropping `String`.

## What I take away from it

1. **Hardware first.** An actuator must be in a safe state before the board knows it has an IP address.
2. **An unstable prop leaves the game.** Safe mode keeps the board reachable but cuts MQTT: a silent prop is better than one that sends phantom events.
3. **Decisions are pure functions.** Reconnection, recovery, configuration choice, port writes: everything that decides is tested on a PC, case by case.
4. **Configuration is data, with a plan B and a plan C.** It's validated, described by schemas, and must never be able to make a board unreachable.
5. **The field decides.** Polling over interrupts, Ethernet over Wi-Fi, debounce, logic inversion by configuration: every choice comes from a real room.
6. **Updating the base breaks no prop.** It's a rule written into every spec, and CI checks it.
7. **Firmware ships like software.** Signed, reproducible, published by CI, updated remotely without touching the configuration.

A good prop is one nobody talks about: the chest opens when it should, the statuette is recognized first time, and the day something goes wrong, the board says so instead of going quiet.
