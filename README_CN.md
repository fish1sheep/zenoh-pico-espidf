# zenoh-pico ESP-IDF 组件

[![许可证](https://img.shields.io/badge/License-EPL%202.0%20OR%20Apache%202.0-blue)](LICENSE.txt)
![版本](https://img.shields.io/badge/version-1.9.0-green)

**zenoh-pico** 是 Eclipse zenoh 协议面向资源受限设备的轻量级 C 语言实现。本仓库将其打包为 **ESP-IDF 组件**，适用于 ESP32 系列微控制器。

zenoh 是一个集发布/订阅 (Pub/Sub) 与查询/存储 (Query/Storage) 于一体的协议，统一了传输中的数据、存储中的数据和计算。它以高效、低延迟和可扩展性著称，是物联网、机器人和边缘计算的理想选择。

> [什么是 zenoh？](https://zenoh.io/) · [zenoh-pico 上游仓库](https://github.com/eclipse-zenoh/zenoh-pico)

---

## 核心特性

- **Zenoh 协议兼容** — 与 Rust zenoh、zenoh-c、zenoh-java、zenoh-python 完全互操作
- **超轻量** — 二进制体积约 50 KB，内存占用仅数十 KB，为 ESP32 优化
- **多传输层支持** — TCP、UDP（单播与多播）、蓝牙、串口、WebSocket、原始以太网
- **完整的通信原语** — 发布/订阅、查询/存储、活跃度检测、管理空间
- **ESP-IDF 原生集成** — 即插即用组件，支持 `menuconfig` 可视化配置
- **双许可证** — EPL-2.0 或 Apache-2.0（任选）

---

## 快速开始

### 前置条件

- [ESP-IDF](https://docs.espressif.com/projects/esp-idf/) v5.x 或更新版本
- ESP32 系列开发板（ESP32、ESP32-S3、ESP32-C3 等）

### 安装

将本仓库克隆到项目的 `components/` 目录下：

```bash
cd your_project/components/
git clone https://github.com/fish1sheep/zenoh-pico-espidf zenoh-pico
```

或者直接将 `zenoh-pico-espidf` 文件夹复制到 `components/` 目录下。

### 最小示例 — 发布

```c
#include <zenoh-pico.h>
#include "freertos/FreeRTOS.h"

void app_main(void) {
    // 创建默认配置
    z_owned_config_t config = z_config_default();
    zp_config_insert(z_loan(config), Z_CONFIG_CONNECT_KEY, "tcp/192.168.1.100:7447");

    // 打开会话
    z_owned_session_t s = z_open(z_move(config));
    if (!z_check(s)) {
        printf("打开 zenoh 会话失败\n");
        return;
    }

    // 在键表达式 "esp32/data" 上声明一个发布者
    z_owned_publisher_t pub = z_declare_publisher(z_loan(s), z_keyexpr("esp32/data"));
    if (!z_check(pub)) {
        printf("声明发布者失败\n");
        return;
    }

    // 定时发布数据
    uint8_t data[] = "Hello from ESP32!";
    while (1) {
        z_publish(z_loan(pub), z_bytes(data, sizeof(data) - 1));
        printf("已发布数据\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 最小示例 — 订阅

```c
#include <zenoh-pico.h>
#include "freertos/FreeRTOS.h"

void callback(z_loaned_sample_t *sample, void *arg) {
    z_owned_string_t s;
    z_bytes_to_string(z_sample_payload(sample), &s);
    printf(">> [订阅者] 收到 %s\n", z_string_data(z_loan(s)));
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

## API 概览

zenoh-pico 提供以下核心原语，所有函数均以 `z_` 或 `zp_` 为前缀。

| 类别 | 关键函数 | 说明 |
|------|----------|------|
| **会话** | `z_open()`, `z_close()` | 建立 / 关闭 zenoh 会话 |
| **配置** | `z_config_default()`, `zp_config_insert()` | 配置端点、传输层、模式 |
| **发布 / 订阅** | `z_declare_publisher()`, `z_declare_subscriber()` | 基础发布-订阅 |
| **高级 Pub / Sub** | `z_declare_advanced_publisher/subscriber()` | 增加拥塞控制、可靠性 |
| **查询 / 存储** | `z_declare_queryable()`, `z_get()` | 请求-应答与可查询存储 |
| **活跃度** | `z_liveliness_declare_token()`, `z_liveliness_declare_subscriber()` | 节点在线检测 |
| **管理** | `zp_admin_space_query()` | 内省与管理 |
| **编码** | `z_encoding()`, `z_bytes()` | 序列化辅助函数 |

完整 API 参考请查阅 [zenoh-pico API 文档](https://github.com/eclipse-zenoh/zenoh-pico)。

---

## 传输层支持

| 传输层 | 状态 | 说明 |
|--------|------|------|
| TCP | ✅ | 默认，测试最充分 |
| UDP 单播 | ✅ | 低延迟，尽最大努力 |
| UDP 多播 | ✅ | 自动发现 / scouting |
| 蓝牙 (ESP32) | ✅ | Arduino ESP32 平台 |
| 串口 (UART) | ✅ | TTY、UART（多平台） |
| WebSocket | ✅ | 客户端模式 |
| TLS | ✅ | 基于 TCP 的加密传输 |
| 原始以太网 | ✅ | 实验性 |

通过 `zp_config_insert()` 设置传输层端点：

```c
zp_config_insert(z_loan(config), Z_CONFIG_CONNECT_KEY, "udp/192.168.1.100:7447");
zp_config_insert(z_loan(config), Z_CONFIG_CONNECT_KEY, "tls/192.168.1.100:7447");
```

---

## 配置

打开 ESP-IDF 配置菜单：

```bash
idf.py menuconfig
```

进入 **Component config → Zenoh-pico configuration** 可调整以下选项：

| 选项 | 默认值 | 说明 |
|------|--------|------|
| 连接端点 | (空) | 默认对端地址，如 `tcp/192.168.1.100:7447` |
| 监听端点 | (空) | 本地监听地址 |
| 模式 | `Peer` | Peer（对等）或 Client（客户端）模式 |
| 最大会话数 | `1` | 最大并发会话数 |
| 最大发布者数 | `64` | 最大声明的发布者数 |
| 最大订阅者数 | `64` | 最大声明的订阅者数 |
| 传输超时 | `10000` | 毫秒 |

也可以在代码中编程配置：

```c
z_owned_config_t config = z_config_default();
zp_config_insert(z_loan(config), Z_CONFIG_CONNECT_KEY, "tcp/192.168.1.100:7447");
zp_config_insert(z_loan(config), Z_CONFIG_MODE_KEY, "client");
```

---

## 构建

标准 ESP-IDF 构建流程：

```bash
idf.py set-target esp32      # 或 esp32s3、esp32c3 等
idf.py build
idf.py flash monitor
```

组件在编译时会自动定义 `ZENOH_ESPIDF` 预处理宏，启用 ESP-IDF 平台层（堆管理、随机数、网络、线程、时间等）。

---

## 参考链接

- [zenoh 官网](https://zenoh.io/)
- [zenoh-pico（上游仓库）](https://github.com/eclipse-zenoh/zenoh-pico)
- [zenoh 文档](https://zenoh.io/docs/)
- [ESP-IDF 编程指南](https://docs.espressif.com/projects/esp-idf/)
- [zenoh-pico-espidf（GitHub）](https://github.com/fish1sheep/zenoh-pico-espidf)

## 许可证

本项目基于 **EPL-2.0 OR Apache-2.0** 双许可证授权 — 详见 [LICENSE.txt](LICENSE.txt)。
