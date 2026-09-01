# 04 · 核心模块逐个讲（功能 / 接口 / 实现 / 依赖 / 边界）

> 基准：`firmware/esp_recorder_v1.2/main/`。每个模块按「功能 → 接口 → 实现 → 依赖 → 边界」五段。

## 1. UART 采集（uart_driver）

- **功能**：两路独立 UART 原始字节采集 + 双向发送 + RX/TX 字节统计。
- **接口**（`uart_driver.h`）：
  - `user_uart_init_channel(ch, baud)` 初始化单通道；
  - `user_uart_configure_channel(ch, baud, databits, stopbits, parity)` 运行时热更新参数；
  - `user_uart_read_channel / write_channel` 读写；
  - `user_uart_get_rx_bytes / get_tx_bytes / channel_is_ready` 状态查询；
  - `user_uart_set_tx_callback(cb)` 注册 TX 回调（把已发送数据回灌 live 流）。
- **实现**：`s_uart_hw[2]` 硬件映射表（CH1=UART0/43/44，CH2=UART2/17/18）；`uart_param_config→uart_set_pin→uart_driver_install`；`portMUX_TYPE s_state_lock` 保护计数器；`uart_is_driver_installed` 区分首装/改参。
- **依赖**：`driver/uart.h`（ESP-IDF UART Driver）。
- **边界**：① 轮询式读（RX task 100ms 超时），非事件/中断驱动；② 两路**共用同一波特率**（`user_uart_init` 遍历两路）；③ CH1 占用 UART0，与控制台 UART0 冲突（见 08）。

## 2. SD 记录（sd_card + data_recorder）

- **功能**：SD 卡 FATFS 挂载 + 双通道原始字节落盘 + 实时流 + 掉电保护。
- **接口**（`sd_card.h` / `data_recorder.h`）：
  - `sd_card_init / unmount_fs / remount_fs / get_handle / get_state`；
  - `recorder_init / start / stop / force_sync / write_rx_channel / write_tx_channel / live_fetch / live_clear`；
  - 统计：`recorder_get_drop_count / get_total_bytes / get_channel_* / get_filename`。
- **实现**：
  - SD：SPI2 + 20MHz + FATFS，`/sdcard`；
  - 落盘：每通道 16KB 环形缓冲 `save_rb`，`recorder_write_raw` 入缓冲，`rec_io` 任务双条件（≥2KB 或 1s）触发 `recorder_flush_channel_locked` → `fwrite`，`rec_sync` 任务每秒 `fsync`；
  - 文件：`/sdcard/REC_CH{n}_YYYYMMDD_HHMMSS[_N].bin`，首个 RX 字节到达才 `fopen(...,"wb")` 懒创建。
- **依赖**：`sdspi_host`、`sdmmc_cmd`、`esp_vfs_fat`、`ff.h`；`data_recorder` 依赖 `sd_card`（查 `sd_card_get_state()`）。
- **边界**：① TX 不落盘（只进 live）；② 文件为 raw `.bin`（无文件头/时间戳）；③ `max_files=5` 上限；④ 无压缩。

## 3. USB 复合设备（usb_cdc + usb_msc）

- **功能**：一根 USB 线同时提供 CDC 虚拟串口 + MSC U 盘读卡器；USB 插拔时与 FATFS 互斥切换 SD 所有权。
- **接口**：
  - `usb_cdc_init(rx_cb)` / `usb_cdc_write / usb_cdc_read`；
  - `usb_msc_init()`（占位）；
  - TinyUSB 静态回调：`tud_msc_inquiry/test_unit_ready/capacity/start_stop/read10/write10/scsi_cb`；
  - `tud_mount_cb / tud_umount_cb`（在 `app_main.c`）。
- **实现**：
  - 描述符：`desc_configuration[]` 并列 CDC 控制接口 + CDC 数据接口 + MSC 接口，设备描述符 `bDeviceProtocol=MISC_PROTOCOL_IAD`（复合），VID=303A/PID=4002；
  - MSC 读写：`sdmmc_read/write_sectors` 直接访问 `sd_card_get_handle()`，3 次重试 + 10ms 延时；
  - 互斥：`tud_mount_cb` → `recorder_stop()` + `sd_card_unmount_fs()`；`tud_umount_cb` → `sd_card_remount_fs()` + 可选 `recorder_start()`。
- **依赖**：`tinyusb.h`、`tusb_cdc_acm.h`、`tusb.h`、`sdmmc_cmd.h`；`sdkconfig` TinyUSB CDC/MSC 均使能。
- **边界**：① CDC 收到数据被 `app_main` 空回调丢弃（未做 USB 配网/命令）；② `usb_msc_init` 无实质逻辑；③ 无 MSC 多 LUN（单 LUN）。

## 4. WiFi / Web（wifi_config + web_server）

- **功能**：AP+STA 混合 WiFi + HTTP REST 管理界面。
- **接口**（`wifi_config.h` / `web_server.h`）：
  - `wifi_config_init / wifi_apply_ap / wifi_connect_sta / wifi_disconnect_sta / wifi_get_state / wifi_get_sta_ip / wifi_get_sta_rssi`；
  - `web_server_start`；22 个 URI handler。
- **实现**：
  - AP：默认 SSID `ESP_Recorder_XXXX`（MAC 后两位）、密码 12345678、IP 192.168.4.1/24、channel 1、max 4、WPA2_PSK；`esp_wifi_set_mode(WIFI_MODE_APSTA)`；
  - STA：断线重连 3 次 × 5s，成功清零；`WIFI_CONNECTED_BIT/WIFI_FAIL_BIT` EventGroup；
  - Web：端口 80，`esp_http_server`，前端 `EMBED_FILES` 内嵌，实时流 200ms 增量轮询。
- **依赖**：`esp_wifi`、`esp_event`、`esp_netif`、`lwip/inet.h`；`esp_http_server`、`cJSON`。
- **边界**：① 无鉴权/无 HTTPS；② 断线重连在默认事件循环里 `vTaskDelay(5000)` 阻塞；③ 无 WebSocket（轮询替代）。

## 5. 时间同步 —— 状态：v1.3 功能，源码缺失

- **功能**：README 称 v1.3_release_test 含"时间同步"（用于给记录文件/实时流打准确时间戳）。
- **现实**：v1.3 目录是空 gitlink；v1.2 中时间来自 `gettimeofday/localtime_r`（`data_recorder.c` L114-L124 `format_iso_timestamp`），**未实现任何外部时间同步（无 SNTP/NTP 代码）**。grep `scripts/` 与 `main/` 均无 `sntp`/`ntp` 调用。
- **结论**：时间同步协议**无法从仓库取证**，标 **【待本人确认】**。v1.2 的记录时间戳依赖上电后系统时钟（ESP32 无 RTC 电池时默认从 1970/构建时间起算），这是可讲的真实边界。

## 6. 安全测试模式 —— 状态：v1.3 功能，源码缺失

- **功能**：README 称 v1.3 含"安全测试模式"。
- **现实**：v1.3 源码缺失，仓库内**无任何"安全测试模式"实现证据**。v1.2 中与"安全"相关的只有：`is_safe_filename`（路径穿越防护）、`valid_sta_authmode`（authmode 校验）、`clamp_*`（参数范围钳制）。
- **结论**：标 **【待本人确认】**，面试切勿把"路径穿越防护"夸大成"安全测试模式"。

## 7. cdc 测试协议 —— 状态：v1.3 功能，源码缺失

- **功能**：README 称 v1.3 含"cdc 测试协议"（推测：通过 USB CDC 虚拟串口下发测试指令、回传结果的协议）。
- **现实**：v1.3 源码缺失；v1.2 的 CDC 只有"收→回调→丢弃"的空闭环（`app_main.c` L24-L28）。无任何测试协议帧定义。
- **结论**：标 **【待本人确认】**。

## 8. usb_cdc_bridge —— 状态：v1.2_dev 功能，源码缺失

- **功能**：README 称 v1.2_dev 新增 `usb_cdc_bridge`（推测：USB CDC ↔ UART 双向桥，让电脑通过虚拟串口直接收发外部设备数据）。
- **现实**：v1.2_dev 目录是空 gitlink；v1.2 中 `usb_cdc_rx_handler` 空实现，**未实现 CDC↔UART 转发**（`HANDOVER_AUDIT_STAGE1.md` L87 明确"USB CDC 收到的数据未解析、未转发"）。
- **结论**：标 **【待本人确认】**。
