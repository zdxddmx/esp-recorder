# 01 · 技术栈（是什么 → 为什么 → 怎么用 → 代码在哪 → 面试官怎么追问）

> 证据约定：所有宏/结构体/函数均给出 `文件:行号`（以 v1.2 为准，v1.2 是仓库中可读的最新完整固件）。

## 0. 总览

| 层 | 技术 | 证据 |
|---|---|---|
| MCU | ESP32-S3（Xtensa LX7 双核 240MHz） | `sdkconfig` `CONFIG_IDF_TARGET="esp32s3"`、`CONFIG_IDF_TARGET_ARCH_XTENSA=y` |
| 框架 | ESP-IDF v5.4（C + FreeRTOS） | `docs/ESP_Recorder_项目开发完整报告.md` L5；`HANDOVER_AUDIT_STAGE1.md` L20 |
| RTOS | FreeRTOS（`CONFIG_FREERTOS_HZ=100`） | `sdkconfig` L1336 |
| 串口 | ESP-IDF UART Driver（2 路） | `uart_driver.c` |
| 存储 | SDSPI（SPI2）+ FATFS | `sd_card.c` |
| USB | TinyUSB（Espressif 封装）CDC + MSC | `usb_cdc.c` / `usb_msc.c` |
| 网络 | esp_wifi（AP+STA）+ esp_http_server | `wifi_config.c` / `web_server.c` |
| 前端 | 纯 HTML/CSS/JS（零依赖，EMBED_FILES 内嵌） | `main/index.html` / `app.css` / `app.js`；`main/CMakeLists.txt` L3 |
| 配置 | NVS（主）+ `/sdcard/config.json`（镜像）+ cJSON | `config_manager.c` |
| 报告 | Python + python-docx（两个报告生成脚本） | `scripts/*.py` |

---

## 1. ESP32-S3（MCU）

- **是什么**：Espressif 双核 Xtensa LX7 @240MHz 的 WiFi+BLE SoC，**带原生 USB OTG**（USB Serial/JTAG + 可编程 USB Device 控制器）。
- **为什么用**：需求要 WiFi + 串口 + USB U 盘，ESP32-S3 是"一颗芯片同时具备 WiFi + 原生 USB OTG"的最低成本选型；USB 可直连 D+/D-（需求 `docs/ESP32.md` L121 "TypeC 直连 D- 和 D+，不要 CH340"）。
- **怎么用**：ESP-IDF `esp32s3` 目标；原生 USB 走 TinyUSB，VBUS 检测用 GPIO48。
- **代码在哪**：`sdkconfig` L370-L374（target/arch）；`usb_cdc.c` L95 `.vbus_monitor_io = GPIO_NUM_48`；`docs/ESP32.md` L116（"ESP32 作为 MCU 使用"）。
- **面试官怎么追问**：
  - "ESP32-S3 和 ESP32 有什么区别？" → 双核 LX7 vs LX6、有原生 USB OTG、有 8MB PSRAM 可选（本工程 sdkconfig `CONFIG_SPIRAM is not set` 实际**未启用 PSRAM**）。
  - "Flash 多大？" → 报告写 16MB，但 **sdkconfig `CONFIG_ESPTOOLPY_FLASHSIZE="2MB"`**——这是文档与配置不一致点，要主动讲（见 07）。

## 2. ESP-IDF（框架）

- **是什么**：Espressif 官方开发框架，基于 CMake + 组件（component）体系，内置 FreeRTOS、驱动、网络栈（lwIP）、HTTP server 等。
- **为什么用**：任务书 `docs/=任务.txt` L40-L42 **明确禁止 Arduino**，要求 "ESP-IDF 官方框架，C 语言 + FreeRTOS，优先使用 esp_wifi/esp_event/nvs_flash/esp_http_server/FATFS/SDMMC 或 SDSPI/UART Driver/TinyUSB/FreeRTOS Queue·EventGroup·Task Notification"。
- **怎么用**：`idf_component.yml` 声明 `idf: >=4.1.0`（实际 v5.4）；`main/CMakeLists.txt` `idf_component_register(...)` 编译 12 个 .c + `EMBED_FILES` 3 个前端文件；依赖组件通过 `dependencies.lock` + `managed_components`/本地 `components/` 提供。
- **代码在哪**：`main/idf_component.yml`；`main/CMakeLists.txt`；顶层 `CMakeLists.txt`。
- **面试官怎么追问**：
  - "为什么不用 Arduino？" → Arduino 抽象层屏蔽了底层（无法精细控制 TinyUSB 描述符、SD 与 USB MSC 的块设备互斥、FreeRTOS 任务优先级），且任务书明文禁止。
  - "组件化结构？" → 任务书 `docs/=任务.txt` L60-L70 建议拆 9 个组件（app_config/wifi_manager/web_server/uart_bridge/data_router/recorder/storage_manager/usb_device/system_status），但**实际实现全部堆在 `main/` 一个组件里**（12 个 .c 并列）——这是"任务书要求 vs 实际落地"的差距点，面试可主动讲并给改进方向。

## 3. FreeRTOS（RTOS）

- **是什么**：抢占式实时内核，任务 + 队列 + 信号量/互斥量 + 事件组 + 通知。
- **为什么用**：UART 接收、SD 落盘、WiFi 事件、USB 协议栈、HTTP 请求天然并发，需要任务隔离；SD 写慢时不能阻塞 UART 收数。
- **怎么用**：5 个业务任务 + 若干框架任务（见 06）；用 `xSemaphoreCreateMutex`（`s_mutex`/`s_live_mutex`）、`xSemaphoreCreateBinary`（`s_data_sem`）、`xEventGroupCreate`（WiFi）、`portMUX_TYPE`（自旋锁保护计数器）。
- **代码在哪**：`app_main.c` L182-L193（xTaskCreate）；`data_recorder.c` L374-L375（信号量）；`wifi_config.c` L22（EventGroup）；`uart_driver.c` L10（portMUX）。
- **面试官怎么追问**："优先级怎么排？为什么 rec_sync 最低？" → 见 06/10。

## 4. UART Driver（串口）

- **是什么**：ESP-IDF `driver/uart.h`，带 1024B 级硬件 FIFO + 驱动层 ring buffer。
- **为什么用**：采集两路外部设备串口数据。
- **怎么用**：CH1=UART0(TX43/RX44)、CH2=UART2(TX17/RX18)，每路 `USER_UART_BUF_SIZE=2048` 驱动缓冲，8N1 默认、115200，运行时可用 `user_uart_configure_channel()` 改波特率/数据位/停止位/校验位。
- **代码在哪**：`uart_driver.h` L16-L24（宏）、L34-L35（configure 接口）；`uart_driver.c` L44-L101（init）、L163-L201（configure）、L203-L237（read/write）。
- **面试官怎么追问**："UART 大数据量怎么处理？会不会丢？" → 见 10。

## 5. SDMMC / SDSPI + FATFS（存储）

- **是什么**：SD 卡访问两套方案——SDMMC（4-bit 高速）与 SDSPI（SPI 总线）。本工程用 **SDSPI**。文件系统 FATFS（`esp_vfs_fat` 挂载到 `/sdcard`）。
- **为什么用 SPI 而非 SDMMC**：报告 L803 "释放 SDMMC 引脚给其他外设，SPI 速率足够记录吞吐"。
- **怎么用**：SPI2_HOST，CS=10/MOSI=38/MISO=40/SCK=39，20MHz，`max_files=5`，`allocation_unit_size=16KB`，`format_if_mount_failed=false`（不自动格式化，保护用户数据）。
- **代码在哪**：`sd_card.c` L16-L22（引脚宏）、L28-L80（init）、L123-L167（unmount/remount）。
- **面试官怎么追问**："SD 写慢怎么办（背压）？" → 16KB 环形缓冲 + 双条件落盘 + drop_count 统计（`data_recorder.c`），见 06/10。

## 6. TinyUSB（USB）

- **是什么**：开源 USB Device 协议栈；ESP-IDF 提供 `espressif__tinyusb`（上游）+ `espressif__esp_tinyusb`（封装层）两个托管组件。
- **为什么用**：实现 **CDC(ACM 虚拟串口) + MSC(大容量存储) 复合设备**，一根 USB 线既当调试串口又当 U 盘。
- **怎么用**：自定义配置描述符 `desc_configuration[]`（CDC 控制接口+数据接口+MSC 接口），VID=0x303A(Espressif) PID=0x4002，IAD 复合（`MISC_PROTOCOL_IAD`），MSC SCSI 回调 `tud_msc_read10_cb/write10_cb` 直接读写 `sdmmc_card_t`，3 次重试。
- **代码在哪**：`usb_cdc.c` L15-L67（描述符）、L85-L122（init）；`usb_msc.c` L41-L161（SCSI 回调）；`sdkconfig` L2080-L2134（TinyUSB Kconfig：MSC/CDC 均使能，DMA 模式，任务优先级 5 / 栈 4096 / CPU1 亲和）。
- **面试官怎么追问**："复合设备怎么枚举？CDC 和 MSC 冲突吗？" → 见 04/10。

## 7. esp_http_server + Web（网络/前端）

- **是什么**：ESP-IDF 内置轻量 HTTP server（`esp_http_server.h`），注册 URI→handler 回调。
- **为什么用**：手机连 AP 热点后浏览器直接管理，无需安装 App。
- **怎么用**：端口 80，v1.2 注册 **22 个路由**；前端 `index.html/app.css/app.js` 用 `EMBED_FILES` 编成 C 数组进固件（`index_handler` 用 `extern _binary_index_html_start[]`）；实时数据用 **200ms 增量轮询**（`/api/stream?since=N`），非 WebSocket。
- **代码在哪**：`web_server.c` L26（`WEB_PORT 80`）、L85-L113（index/css/js handler）、L776-L797（路由表）；`main/CMakeLists.txt` L3（EMBED_FILES）。
- **面试官怎么追问**："为什么不用 WebSocket？为什么不用 HTTPS？" → 见 10/13。

## 8. NVS + cJSON（配置）

- **是什么**：NVS 是 ESP32 Flash 上的键值存储；cJSON 是轻量 JSON 解析/拼装库。
- **为什么用**：v1.1 早期配置只放 SD 卡（`/sdcard/config.json`），v1.2 改为 **NVS 为主（namespace `recorder_cfg`，key `config_json`），SD config.json 为可读镜像**（拔卡也能读配置；用户仍可用电脑直接改镜像）。
- **怎么用**：`nvs_set_str/nvs_get_str` + `cJSON_Parse/cJSON_Print`；字段校验（`clamp_authmode`、`clamp_stream_gap_ms`、`valid_baudrate` 1200~2M）、写失败回滚（`save_or_rollback`）。
- **代码在哪**：`config_manager.c` L13-L16（NVS 命名空间/key）、L172-L215（NVS 读写）、L287-L330（load：NVS 优先 → SD 兜底 → 迁移）、L376-L385（回滚）。
- **面试官怎么追问**："配置放 NVS 还是 SD？为什么？掉电安全吗？" → 见 10。

## 9. Flutter（App）—— 状态：仓库内无源码

- **是什么**：README 宣称配套 Flutter 手机/桌面 App。
- **现实**：`app/esp_recorder_app_v1.0` 是**空 gitlink**（`git ls-files -s` mode `160000`，`.gitmodules` 缺失），仓库内**没有 Flutter 源码**；`docs/FLUTTER_ENV_SETUP.md` L127 自述"本次仅完成环境检查与文档记录；**未创建 Flutter 项目**"。
- **结论**：面试若被问 App，如实说"App 侧我主要负责环境评估与文档（`docs/FLUTTER_ENV_SETUP.md`），Flutter 工程源码未随仓库提交"，标 **【待本人确认】**。

## 10. Python 报告生成脚本

- **是什么**：`scripts/build_report_docx.py`（1212 行）与 `scripts/build_tech_report.py`（1469 行），均用 **python-docx** 生成 Word 报告。
- **为什么用**：把"源码 + 逐行拆解 + 设计决策 + 接线 + 构建 + 排障"固化成交付文档（面向客户/验收）。
- **怎么用**：脚本内嵌 `SRC = {}` / `CODE_*` 字符串常量（模块源码摘录）+ 辅助函数（`add_table/add_code_block/add_photo_placeholder/…`）+ `build()/build_docx()` 主流程，末尾 `doc.save(路径)`。
- **代码在哪**：`scripts/build_report_docx.py` L564-L658（辅助函数）、L662-L1212（主构建）；`scripts/build_tech_report.py` L929-L1006（辅助函数）、L1010-L1469（主构建）。
- **面试官怎么追问**："为什么用脚本生成报告而不是手写？" → 内容与代码强绑定、可重复生成、便于改代码后一键更新交付物；本质是"用脚本固化交付流程"的工程化意识。
