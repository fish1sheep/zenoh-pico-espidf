# zenoh-pico ESP-IDF Component

[![License](https://img.shields.io/badge/License-EPL%202.0%20OR%20Apache%202.0-blue)](LICENSE.txt)
![Version](https://img.shields.io/badge/version-1.9.0-green)

**zenoh-pico** is the Eclipse zenoh implementation targeting constrained devices, offering a native C API. This repository packages it as an **ESP-IDF component** for ESP32 series microcontrollers.

zenoh is a pub/sub/query protocol that unifies data in motion, data at rest, and computations. It is designed to be efficient, low-latency, and scalable — ideal for IoT, robotics, and edge computing.

> [What is zenoh?](https://zenoh.io/) · [zenoh-pico upstream](https://github.com/eclipse-zenoh/zenoh-pico)

---

## Features

- **Zenoh protocol compatible** — fully interoperable with Rust zenoh, zenoh-c, zenoh-java, zenoh-python
- **Ultra-lightweight** — ~50 KB binary footprint, tens-of-KB RAM usage, ideal for ESP32
- **Multiple transports** — TCP, UDP (unicast & multicast), Bluetooth, Serial, WebSocket, Raw Ethernet
- **Full communication primitives** — Pub/Sub, Query/Storage, Liveliness, Admin space
- **ESP-IDF native** — drop-in component, `menuconfig` integration
- **Dual license** — EPL-2.0 OR Apache-2.0

---

## Quick Start

### Prerequisites

- [ESP-IDF](https://docs.espressif.com/projects/esp-idf/) v5.x or later
- ESP32 series board (ESP32, ESP32-S3, ESP32-C3, etc.)

### Installation

Clone this repository into your project's `components/` directory:

```bash
cd your_project/components/
git clone https://github.com/fish1sheep/zenoh-pico-espidf zenoh-pico
```

Alternatively, copy the `zenoh-pico-espidf` folder directly into `components/`.

### Minimal Example — Publish

```c
#include <zenoh-pico.h>
#include "freertos/FreeRTOS.h"

void app_main(void) {
    // Initialize a default zenoh session configuration
    z_owned_config_t config = z_config_default();
    zp_config_insert(z_loan(config), Z_CONFIG_CONNECT_KEY, "tcp/192.168.1.100:7447");

    // Open the session
    z_owned_session_t s = z_open(z_move(config));
    if (!z_check(s)) {
        printf("Failed to open zenoh session\n");
        return;
    }

    // Declare a publisher on key expression "esp32/data"
    z_owned_publisher_t pub = z_declare_publisher(z_loan(s), z_keyexpr("esp32/data"));
    if (!z_check(pub)) {
        printf("Failed to declare publisher\n");
        return;
    }

    // Publish periodically
    uint8_t data[] = "Hello from ESP32!";
    while (1) {
        z_publish(z_loan(pub), z_bytes(data, sizeof(data) - 1));
        printf("Published data\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### Minimal Example — Subscribe

```c
#include <zenoh-pico.h>
#include "freertos/FreeRTOS.h"

void callback(z_loaned_sample_t *sample, void *arg) {
    z_owned_string_t s;
    z_bytes_to_string(z_sample_payload(sample), &s);
    printf(">> [Subscriber] Received %s\n", z_string_data(z_loan(s)));
    z_string_drop(z_move(s));
}

void app_main(void) {
    z_owned_config_t config = z_config_default();
    zp_config_insert(z_loan(config), Z_CONFIG_CONNECT_KEY, "tcp/192.168.1.100:7447");

    z_owned_session_t s = z_open(z_move(config));
    if (!z_check(s)) return;

    z_owned_subscriber_t sub = z_declare_subscriber(
        z_loan(s), z_keyexpr("esp32/+/data"), z_move(callback), NULL);
    if (!z_check(sub)) return;

    while (1) { vTaskDelay(pdMS_TO_TICKS(1000)); }
}
```

---

## API Overview

zenoh-pico provides the following core primitives. All functions are prefixed with `z_` or `zp_`.

| Category | Key Functions | Description |
|----------|--------------|-------------|
| **Session** | `z_open()`, `z_close()` | Establish / close a zenoh session |
| **Configuration** | `z_config_default()`, `zp_config_insert()` | Configure endpoints, transport, mode |
| **Pub / Sub** | `z_declare_publisher()`, `z_declare_subscriber()` | Basic publish-subscribe |
| **Advanced Pub / Sub** | `z_declare_advanced_publisher/subscriber()` | Add congestion control, reliability |
| **Query / Storage** | `z_declare_queryable()`, `z_get()` | Request-reply and queryable storage |
| **Liveliness** | `z_liveliness_declare_token()`, `z_liveliness_declare_subscriber()` | Peer presence detection |
| **Admin** | `zp_adminspace_query()` | Introspection and management |
| **Encoding** | `z_encoding()`, `z_bytes()` | Serialization helpers |

For the complete API reference, see the [zenoh-pico API documentation](https://github.com/eclipse-zenoh/zenoh-pico).

---

## Transport Support

| Transport | Status | Notes |
|-----------|--------|-------|
| TCP | ✅ | Default, most widely tested |
| UDP Unicast | ✅ | Low-latency, best-effort |
| UDP Multicast | ✅ | Auto-discovery / scouting |
| Bluetooth (ESP32) | ✅ | Arduino ESP32 platform |
| Serial (UART) | ✅ | TTY, UART on various platforms |
| WebSocket | ✅ | Client mode |
| TLS | ✅ | Encrypted over TCP |
| Raw Ethernet | ✅ | Experimental |

Set the transport endpoint via `zp_config_insert()`:

```c
zp_config_insert(z_loan(config), Z_CONFIG_CONNECT_KEY, "udp/192.168.1.100:7447");
zp_config_insert(z_loan(config), Z_CONFIG_CONNECT_KEY, "tls/192.168.1.100:7447");
```

---

## Configuration

Open the ESP-IDF configuration menu:

```bash
idf.py menuconfig
```

Navigate to **Component config → Zenoh-pico configuration** to adjust:

| Option | Default | Description |
|--------|---------|-------------|
| Connect endpoint | `(empty)` | Default peer address e.g. `tcp/192.168.1.100:7447` |
| Listen endpoint | `(empty)` | Address to listen on |
| Mode | `Peer` | Peer or Client mode |
| Maximum sessions | `1` | Max concurrent sessions |
| Maximum publishers | `64` | Max declared publishers |
| Maximum subscribers | `64` | Max declared subscribers |
| Transmission timeout | `10000` | ms |

You can also configure programmatically:

```c
z_owned_config_t config = z_config_default();
zp_config_insert(z_loan(config), Z_CONFIG_CONNECT_KEY, "tcp/192.168.1.100:7447");
zp_config_insert(z_loan(config), Z_CONFIG_MODE_KEY, "client");
```

---

## Building

Standard ESP-IDF build process:

```bash
idf.py set-target esp32      # or esp32s3, esp32c3, etc.
idf.py build
idf.py flash monitor
```

The component is automatically compiled with the `ZENOH_ESPIDF` preprocessor flag, enabling the ESP-IDF platform layer (heap management, random, networking, threading, time).

---

## References

- [zenoh home page](https://zenoh.io/)
- [zenoh-pico (upstream)](https://github.com/eclipse-zenoh/zenoh-pico)
- [zenoh documentation](https://zenoh.io/docs/)
- [ESP-IDF Programming Guide](https://docs.espressif.com/projects/esp-idf/)
- [zenoh-pico-espidf (GitHub)](https://github.com/fish1sheep/zenoh-pico-espidf)

## License

This project is licensed under **EPL-2.0 OR Apache-2.0** — see [LICENSE.txt](LICENSE.txt) for details.
