# 07 · 硬件交互（引脚 / 总线 / 枚举 / 射频，对应代码位置）

> 引脚与总线参数全部从 `uart_driver.h`、`sd_card.c`、`usb_cdc.c`、`wifi_config.c`、`sdkconfig`、`docs/ESP32.md` 取证。⚠️ 原理图不在仓库，GPIO16 的作用等标注"注释声称 / 待确认"。

## 一、引脚总表（从代码取证）

| 功能 | GPIO | 方向 | 证据 |
|---|---|---|---|
| CH1 UART TX | 43 | 输出 | `uart_driver.h` L17 `USER_UART1_TX_PIN 43` |
| CH1 UART RX | 44 | 输入 | `uart_driver.h` L18 |
| CH2 UART TX | 17 | 输出 | `uart_driver.h` L21 `USER_UART2_TX_PIN 17` |
| CH2 UART RX | 18 | 输入 | `uart_driver.h` L22 |
| SD CS | 10 | 输出 | `sd_card.c` L18 `SD_PIN_CS 10` |
| SD MOSI | 38 | 输出 | `sd_card.c` L19 |
| SD MISO | 40 | 输入 | `sd_card.c` L20 |
| SD SCK | 39 | 输出 | `sd_card.c` L21 |
| USB VBUS 检测 | 48 | 输入 | `usb_cdc.c` L95 `.vbus_monitor_io = GPIO_NUM_48` |
| UART 接收使能 | 16 | 输出(低) | `app_main.c` L73-L81 `gpio_set_level(GPIO16, 0)` |

- CH1 用 **UART0**、CH2 用 **UART2**（`uart_driver.h` L16/L20）。
- ⚠️ **GPIO16 的作用只有代码注释**："拉低用于开启串口（接收使能，boot 期间保持）"（`app_main.c` L81），`HANDOVER_AUDIT_STAGE1.md` L77 明说"没有原理图证据"。**原理图未入仓库 → GPIO16 实际作用标【待本人确认】**。

## 二、UART（波特率 / 缓冲 / 模式）

- **模式**：8N1（`UART_DATA_8_BITS` + `UART_PARITY_DISABLE` + `UART_STOP_BITS_1`），无硬件流控（`UART_HW_FLOWCTRL_DISABLE`），`UART_SCLK_DEFAULT` 时钟源（`uart_driver.c` L52-L58）。
- **默认波特率**：115200（`app_main.c` L22 `DEFAULT_BAUD_RATE 115200`；`config_manager.c` L273 默认）。
- **驱动缓冲**：每路 `USER_UART_BUF_SIZE=2048`（RX+TX 各 2048，`uart_driver_install(hw->port, 2048, 2048, ...)`，`uart_driver.c` L74）。
- **运行时可配**：波特率 1200~2000000（`config_manager.c` L371-L374）、数据位 5/6/7/8（`map_data_bits` L141-L149）、停止位 1/2（`map_stop_bits`）、校验 N/O/E（`map_parity`）。
- **读模型**：RX task 每 100ms 超时读最多 256 字节（`app_main.c` L60）。

## 三、SD 总线（SDSPI）

- **总线**：SPI2_HOST（`sd_card.c` L17 `SD_HOST SPI2_HOST`）。
- **速率**：20MHz（`host.max_freq_khz = 20000`，L55）。
- **单次传输上限**：4000 字节（`SD_MAX_TRANSFER_SZ 4000`，L22）。
- **FATFS 挂载**：`/sdcard`，`format_if_mount_failed=false`（不自动格式化）、`max_files=5`、`allocation_unit_size=16*1024`（L61-L65）。
- **为什么 SPI 不用 SDMMC**：`docs/ESP_Recorder_项目开发完整报告.md` L803 "释放 SDMMC 引脚给其他外设"。

## 四、USB（枚举 / 复合设备）

- **接口**：ESP32-S3 原生 USB OTG（需求 `docs/ESP32.md` L121 "TypeC 直连 D- 和 D+，不要 CH340"）。
- **枚举**：CDC(ACM 虚拟串口) + MSC(大容量存储) **复合设备**，一个配置描述符内 `ITF_NUM_CDC`（控制）+ `ITF_NUM_CDC_DATA`（数据）+ `ITF_NUM_MSC` 三个接口（`usb_cdc.c` L15-L41）。
- **端点**：CDC 通知 EP 0x81(8B)、CDC 数据 0x02/0x82(64B)、MSC 0x03/0x83(64B/HS 512B)（L22-L28）。
- **VID/PID**：0x303A(Espressif)/0x4002（L51-L52）。
- **IAD 复合**：`bDeviceProtocol=MISC_PROTOCOL_IAD`（L49）。
- **VBUS**：GPIO48（L95）。
- **TinyUSB 配置**（sdkconfig）：`CONFIG_TINYUSB_RHPORT_FS=y`（Full-Speed）、`CONFIG_TINYUSB_MODE_DMA=y`、MSC BUFSIZE 512、CDC RX/TX BUFSIZE 512（L2081-L2134）。
- **字符串**：`"Espressif" / "ESP Recorder" / "123456" / "ESP Recorder CDC" / "ESP Recorder MSC"`（L60-L67）。

## 五、WiFi 射频

- **模式**：802.11 b/g/n，AP+STA 混合（`esp_wifi_set_mode(WIFI_MODE_APSTA)`，`wifi_config.c` L167）。
- **AP**：channel 1、max_connection 4、WPA2_PSK（或空密码→OPEN），SSID `ESP_Recorder_XXXX`（MAC 后两位）、IP 192.168.4.1/24（L152-L166）。
- **STA**：支持 OPEN/WPA/WPA2/WPA3/WPA2_WPA3（`clamp_authmode`，`config_manager.c` L20-L32）。
- **RSSI 查询**：`wifi_get_sta_rssi`（`wifi_config.c` L269-L278）。

## 六、Flash / 分区（关键不一致点）

| 项 | 报告/文档声称 | sdkconfig 实际 | 说明 |
|---|---|---|---|
| Flash 容量 | 16MB（`docs/...报告.md` L322） | `CONFIG_ESPTOOLPY_FLASHSIZE="2MB"`（L531） | **不一致** |
| 分区表 | 自定义 OTA（factory 4MB + ota_0/1） | `CONFIG_PARTITION_TABLE_SINGLE_APP=y`（L545，用 `partitions_singleapp.csv`） | `partitions.csv` 未启用 |
| PSRAM | "8MB"（报告 L1083） | `CONFIG_SPIRAM is not set`（L1054） | **未启用 PSRAM** |
| 优化等级 | — | `CONFIG_COMPILER_OPTIMIZATION_DEBUG=y`（L559） | Debug 未优化 |

- 证据链：`firmware/esp_recorder_v1.2/sdkconfig` 上述行；`HANDOVER_AUDIT_STAGE1.md` L26 早指出"分区配置不一致"。
- **面试要点**：主动指出"文档里的 16MB 是目标量产配置，仓库 sdkconfig 是 2MB 调试配置"，标 **【待本人确认】**（是否量产板用 16MB）。

## 七、硬件接线注意（报告原文要点）

- 用开发板的 **USB OTG 口**（非 UART 口）连接电脑才能枚举 USB 复合设备（`docs/...报告.md` L145）。
- USB-TTL 模块需**交叉 TX/RX、共地、3.3V 电平、不要给 VCC 供电**（L145）。
- SD 卡 VCC 3.3V 勿接 5V（`build_report_docx.py` L780）。

## 八、硬件验证证据边界

- ✅ 有截图：`screenshots/V1.2_USB_CDC_MSC_硬件验证成功.png`（文件名即"硬件验证成功"）；多张 `微信图片_2026*` 疑似现场照片（内容未逐张核验，标【待本人确认】）。
- ✅ **部分实机验证通过（【作者实习记录】8.4）**：SD 卡 SPI 挂载 ✓、WiFi AP 热点 ✓、Web 管理页面 ✓、数据记录引擎双条件落盘 + fsync 实机运行正常 ✓、USB 复合设备热插拔切换正常 ✓。
- ✅ **路由丢包实测（【作者实习记录】7.30）**：CH2→CH1 路由 1 字节丢包 0 验证通过。
- ⚠️ 报告 12.2 的"待硬件验证"清单（`docs/...报告.md` L720-L727）与 `HANDOVER_AUDIT_STAGE1.md`"未烧录"、`FULL_FEATURES_README.md`"没有执行烧录"均为 **8.3 及更早表述**，被作者 7.29（烧录）、8.4（实机验证）更晚记录覆盖。
- ⚠️ 作者 8.4 记录"双路串口端到端验证持续中" → 最终端到端状态标【待本人确认】。
- ⚠️ 无原理图、无 BOM、无实测吞吐/丢包率曲线（仅有 7.30 的"1 字节路由丢包 0"单点数据）→ **严禁编造吞吐/丢包率**。
- ⚠️ 无原理图、无 BOM、无实测吞吐/丢包数据 → **严禁编造性能数据**。
