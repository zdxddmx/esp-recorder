# 03 · 代码走读（最值得讲的 20 个文件 + 现场演示点击顺序）

> 走读基准版本：`firmware/esp_recorder_v1.2/`（仓库中可读的最新完整固件）。
> ⚠️ v1.2_dev 与 v1.3_release_test 源码是空 gitlink，不在仓库内，无法走读（见 `14_DEVELOPMENT_HISTORY.md`）。

## 0. GitHub 现场演示点击顺序（5 分钟路线，见 12_CODE_NAVIGATION.md 展开）

1. **先讲版本演进目录**：打开 `firmware/`，点出 6 个版本目录，一句话各是什么 → 点 `README.md` L24-L26「版本沿革」。
2. **主入口**：`firmware/esp_recorder_v1.2/main/app_main.c`（讲初始化顺序 + USB 热插拔回调）。
3. **最核心模块**：`main/data_recorder.c`（环形缓冲 + 双条件落盘 + fsync + live 流）。
4. **UART 驱动**：`main/uart_driver.h` + `uart_driver.c`（引脚宏 + 热配置）。
5. **SD + USB**：`main/sd_card.c`（FATFS↔MSC 互斥）→ `main/usb_cdc.c`（复合描述符）→ `main/usb_msc.c`（SCSI）。
6. **WiFi + Web**：`main/wifi_config.c`（AP+STA）→ `main/web_server.c`（22 路由）。
7. **配置**：`main/config_manager.c`（NVS + cJSON + 回滚）。
8. **需求→交付**：`docs/ESP32.md`（需求）→ `docs/=任务.txt`（任务书）→ `docs/ESP_Recorder_项目开发完整报告.md`（交付）。
9. **工具链**：`scripts/build_tech_report.py`（报告生成器）。

---

## 1. `main/app_main.c`（201 行）— 主入口

- **核心函数**：`app_main()`（L71）、`uart_rx_task()`（L53）、`tud_mount_cb()`（L32）、`tud_umount_cb()`（L40）、`usb_cdc_rx_handler()`（L24，空实现占位）。
- **设计原因**：① 初始化顺序=依赖顺序（SD 先于 config，因为 config.json 在 SD；USB 先于 WiFi，尽快暴露调试口）；② 每个子系统失败**降级不 abort**（SD 没插也能用 WiFi/Web）；③ UART 逐通道独立初始化，一通道坏不影响另一通道。
- **改进空间**：`usb_cdc_rx_handler` 空实现（USB 串口收到数据被丢弃，未做 USB 命令解析/配网闭环——`HANDOVER_AUDIT_STAGE1.md` L40 已指出）。
- **面试亮点**：对比 v1.1，讲"从 ESP_ERROR_CHECK 硬 abort 到容错降级"。

## 2. `main/uart_driver.h`（51 行）— UART 引脚/接口定义

- **核心宏**：`USER_UART1_NUM=UART_NUM_0`、`TX=43/RX=44`；`USER_UART2_NUM=UART_NUM_2`、`TX=17/RX=18`；`USER_UART_BUF_SIZE=2048`（L16-L24）。
- **核心接口**：`user_uart_init_channel`、`user_uart_configure_channel`（L34-L35，完整参数热更新）、`user_uart_get_rx/tx_bytes`。
- **设计原因**：引脚用宏集中定义，改硬件只改一处。
- **改进空间**：**CH1 用 UART0，与控制台 UART0 冲突**（sdkconfig `CONFIG_ESP_CONSOLE_UART_NUM=0`）——这是真实风险，见 08/13。

## 3. `main/uart_driver.c`（283 行）— UART 驱动实现

- **核心函数**：`user_uart_init_channel()`（L44，`uart_param_config`+`uart_set_pin`+`uart_driver_install`）、`user_uart_configure_channel()`（L163，`map_data_bits/map_stop_bits/map_parity` 数字→枚举）、`user_uart_write_channel()`（L203，TX 回调）、`user_uart_read_channel()`（L223）。
- **设计原因**：① `uart_is_driver_installed` 区分"首装/改参"；② `portMUX_TYPE s_state_lock` 保护 RX/TX 计数器（多任务线程安全）；③ `user_uart_get_hw()` 做 channel 越界校验（v1.2 加固）。
- **改进空间**：读超时 100ms 的轮询式读（`uart_rx_task` L60 `timeout=100`），非事件驱动；两路共用同一波特率配置（`user_uart_init` 遍历两路同 baud）。

## 4. `main/sd_card.c`（177 行）— SD SPI + FATFS

- **核心函数**：`sd_card_init()`（L28）、`sd_card_unmount_fs()`（L123）、`sd_card_remount_fs()`（L153）、`sd_card_deinit()`（L98）、`sd_card_get_handle()`（L169）。
- **核心结构**：`sd_card_state_t`（`sd_card.h`，含 `mounted`、`usb_owned`、`mount_point`）。
- **设计原因**：① SPI2 + 20MHz（`max_freq_khz=20000`）；② `format_if_mount_failed=false` 保护用户数据；③ **unmount 只卸 FATFS/VFS、保留 `sdmmc_card_t` 给 USB MSC**（L133-L150）；④ remount 先 `sd_card_release_host()` 完全释放再重挂（USB 已释放硬件）。
- **改进空间**：`max_files=5`（FATFS 同时打开文件上限，若 Web 下载 + 双路记录 + config 同时打开可能顶满）；无 SD 卡热插拔检测（只靠 USB 事件）。

## 5. `main/usb_cdc.c`（149 行）— USB CDC + 复合描述符

- **核心函数**：`usb_cdc_init()`（L85）、`usb_cdc_rx_callback()`（L69）、`usb_cdc_write()/read()`（L124/L137）。
- **核心数据**：`desc_configuration[]`（L30，CDC 控制接口+数据接口+MSC 接口并列）、`descriptor_config`（L43，VID=0x303A/PID=0x4002，`bDeviceProtocol=MISC_PROTOCOL_IAD` 复合）、`s_string_desc[]`（L60，6 个字符串描述符）。
- **设计原因**：复合设备靠"配置描述符里接口并列 + IAD"实现；VBUS 检测 GPIO48；`callback_rx` 把收到的数据交给注册的回调（当前 `app_main` 里回调是空实现）。
- **面试亮点**：**这里是复合设备枚举的核心证据**——`TUD_CDC_DESCRIPTOR` + `TUD_MSC_DESCRIPTOR` 同在一个 config。

## 6. `main/usb_msc.c`（189 行）— USB MSC（U 盘）

- **核心函数**：`tud_msc_inquiry_cb`（L41）、`tud_msc_capacity_cb`（L61）、`tud_msc_read10_cb`（L109）、`tud_msc_write10_cb`（L136）、`tud_msc_scsi_cb`（L163）、`usb_msc_init`（L184，占位）。
- **设计原因**：① MSC 回调直接 `sdmmc_read/write_sectors` 访问 `sd_card_get_handle()` 的块设备；② **3 次重试 + 10ms 延时**（`MSC_IO_RETRY_MAX=3`）提升 SPI SD 偶发错误下的可靠性；③ `offset!=0 || bufsize%sector_size!=0` 拒绝非对齐请求（L120/L147）。
- **改进空间**：`usb_msc_init` 只是"占位入口"（L184-L189 仅打日志，真实逻辑全在静态 TinyUSB 回调里）；`tud_msc_scsi_cb` 对非标准 SCSI 命令只回 ILLEGAL_REQUEST（L172-L177）。

## 7. `main/wifi_config.c`（279 行）— WiFi AP+STA

- **核心函数**：`wifi_config_init()`（L80）、`wifi_apply_ap()`（L125）、`wifi_connect_sta()`（L184）、`wifi_event_handler()`（L37）、`wifi_get_sta_ip/rssi`（L255/L269）。
- **核心宏**：`STA_MAX_RETRY=3`、`STA_RETRY_INTERVAL_MS=5000`（L18-L19）。
- **设计原因**：① AP 默认 SSID=`ESP_Recorder_` + MAC 后两位（`wifi_ap_default_ssid` L28）；② 空密码=开放热点；③ STA 断线重连 3 次后放弃（不退出 AP+STA 模式）；④ `wifi_connect_sta` 处理 `ESP_ERR_WIFI_STATE` 用"stop→set_config→start"而非 abort（L219-L227）。
- **改进空间**：`wifi_event_handler` 里 `vTaskDelay(5000)` **阻塞默认事件循环**（L51，`HANDOVER_AUDIT_STAGE1.md` L120 已标为并发/阻塞风险）——应改成定时器或独立任务。

## 8. `main/data_recorder.c`（645 行）— ★ 最核心：记录引擎

- **核心函数**：`recorder_init()`（L368）、`recorder_start()`（L514）、`recorder_stop()`（L554）、`recorder_write_rx/tx_channel`（L578/L589）、`recorder_io_task`（L308）、`recorder_sync_task`（L334）、`recorder_live_fetch`（L609）、`recorder_flush_channel_locked`（L181）。
- **核心宏**：`SAVE_RINGBUF_SIZE=16384`、`SAVE_THRESHOLD_BYTES=2048`、`SAVE_TIMEOUT_MS=1000`、`LIVE_CHUNK_BYTES=48`、`RECORDER_LIVE_CAPACITY=256`（L20-L29）。
- **设计原因（六层，可直接当面试讲稿）**：
  1. 环形缓冲攒批，避免逐字节写 SD；
  2. 双条件落盘（≥2KB 或 1s 超时）；
  3. 每秒 fsync 兜底 + 连续 3 次失败关文件；
  4. live 流与落盘分离缓冲；
  5. 双通道完全解耦（各自缓冲/文件/统计）；
  6. 文件名时间戳 + 同名加 `_N` 后缀防冲突（`recorder_compute_filename_locked` L146）。
- **改进空间**：`recorder_live_fetch` 在持锁下做 O(capacity) 线性扫描 + 逐条 memcpy（L620-L632），高频轮询下锁竞争；`recorder_write_raw` 里 `xSemaphoreGive(s_data_sem)` 在**释放 s_mutex 之后**（L304-L305），存在信号量计数溢出风险（每写一次 give 一次，即使 io task 还没 take）。

## 9. `main/data_router.c`（87 行）— 跨通道路由

- **核心函数**：`data_router_forward()`（L34）、`data_router_set_enabled()`（L26）、`data_router_get_stats()`（L78）。
- **设计原因**：CH1 RX→CH2 TX、CH2 RX→CH1 TX 透明转发；`portMUX_TYPE s_stats_lock` 保护统计；丢包统计 `ch1_to_ch2_drop_bytes` 等（L60-L67）。
- **改进空间**：转发是"读多少写多少"的同步阻塞（`user_uart_write_channel`），目标通道 TX 缓冲满时会拖慢源通道的 RX task。

## 10. `main/web_server.c`（814 行）— HTTP REST API

- **核心函数**：`is_safe_filename()`（L50，路径穿越防护）、`status_handler()`（L115，~40 字段 JSON）、`index/css/js_handler`（L85-L113，EMBED_FILES）、`send_tx_handler`（L565）、`file_download_handler`（L465，分块传输）、`web_server_start`（L764-L805）。
- **核心宏**：`WEB_PORT=80`、`FW_VERSION="v1.2.0"`（L26-L27）。
- **路由表（22 个，L776-L797）**：`/`、`/app.css`、`/app.js`、`/api/status`、`/api/wifi`、`/api/wifi/scan`、`/api/wifi/disconnect`、`/api/wifi/status`、`/api/ap`、`/api/files`、`/api/file`、`/api/file/delete`、`/api/recorder/start`、`/api/recorder/stop`、`/api/stream`、`/api/stream/clear`、`/api/send`、`/api/reboot`、`/api/config/stream_gap`、`/api/config/uart`、`/api/config/router`、`/api/config/recorder`。
- **设计原因**：① cJSON 拼装 JSON（SSID 特殊字符安全）；② 下载用 `httpd_resp_send_chunk` 分块 + `Content-Disposition: attachment`；③ 文件操作用 `is_safe_filename` 拒绝 `/ \ : ..`；④ `file_api_available = sd.mounted && !recording`（记录期间禁止文件操作，`status_handler` L139-L140）。
- **改进空间**：无鉴权（局域网裸奔，`docs/ESP_Recorder_项目开发完整报告.md` L734 自述"仅 HTTP 明文"）；无 WebSocket（任务书要求 /ws，实际 200ms 轮询）。

## 11. `main/config_manager.c`（424 行）— 配置管理器

- **核心函数**：`config_manager_load()`（L287）、`config_manager_save()`（L332）、`parse_json()`（L76）、`build_json()`（L134）、`load_nvs_json/save_nvs_json`（L172/L202）、`save_or_rollback()`（L376）。
- **核心宏**：`CONFIG_NVS_NAMESPACE="recorder_cfg"`、`CONFIG_NVS_KEY="config_json"`、`CONFIG_SCHEMA_VERSION=2`（L13-L16）。
- **设计原因**：① NVS 主存储 + SD config.json 镜像（L287-L330：NVS 优先→SD 兜底→自动迁移）；② 字段校验（`clamp_authmode` L20、`valid_baudrate` L371 1200~2M）；③ 写失败回滚（`save_or_rollback` L376-L385）。
- **改进空间**：WiFi 密码明文存 NVS/config.json（无加密，`docs/=任务.txt` L168 只要求"不写死，存 NVS"，未要求加密）。

## 12. `main/console_cmd.c`（91 行）— 调试控制台

- **核心函数**：`console_start()`（L73）、`cmd_sd_test()`（L25）、`register_sd_test()`（L62）。
- **设计原因**：ESP-IDF Console REPL，注册 `sd_test` 命令测 SD 速度。
- **⚠️ 真实风险**：L13-L14 注释说控制台在 **UART1(TX17/RX18)**，但代码用 `ESP_CONSOLE_DEV_UART_CONFIG_DEFAULT()`（L80）读的是 `CONFIG_ESP_CONSOLE_UART_NUM=0`（UART0），而 **CH1 就是 UART0**——注释、sdkconfig、实际三处不一致，`console_start` 可能因 UART0 已被 CH1 占用而失败/中止。证据：`uart_driver.h` L16、`sdkconfig` `CONFIG_ESP_CONSOLE_UART_NUM=0`、`console_cmd.c` L13/L80。

## 13. `main/sd_speed_test.c`（184 行）— SD 测速工具

- **核心函数**：`sd_speed_test_once()`（L49）、`fill_pattern/verify_pattern`（L21/L29，LCG 确定性伪随机）。
- **设计原因**：用 `esp_timer_get_time` 计时 `fwrite/fread` + `fclose` 触发真正落盘，测 FATFS 层真实吞吐；缓冲区优先 PSRAM（L64 `heap_caps_malloc(MALLOC_CAP_SPIRAM...)`）再回落 heap。
- **改进空间**：测的是"顺序写"最理想情况，未模拟"边收边写 + 掉电"等真实记录场景。

## 14. `main/index.html` / `app.css` / `app.js`（前端 SPA）

- **核心 JS**：`pollStream()`（200ms 增量拉 `/api/stream?since=N`）、`refreshStatus()`（2s）、`sendTx()`、`trimLines()`（限 500 行）。
- **设计原因**：零外部依赖（无 CDN，`docs/=任务.txt` L184-L187 要求）；`EMBED_FILES` 内嵌，固件自包含；本地预览模式用模拟数据（`HANDOVER_AUDIT_STAGE1.md` L102-L105 指出模拟数据不能当硬件证据）。
- **证据**：`web_server.c` L87 `_binary_index_html_start[]`；`main/CMakeLists.txt` L3。

## 15. `main/CMakeLists.txt` + `idf_component.yml`

- `idf_component_register(SRCS 12×.c ... EMBED_FILES "index.html" "app.css" "app.js")`（L1-L3）。
- `idf_component.yml` 只声明 `idf: >=4.1.0`（L4-L5），未声明 cJSON（cJSON 由 IDF 自带）。

## 16. `sdkconfig` — 关键配置取证点

- `CONFIG_IDF_TARGET="esp32s3"`（L372）、`CONFIG_ESPTOOLPY_FLASHSIZE="2MB"`（L531）、`CONFIG_PARTITION_TABLE_SINGLE_APP=y`（L545）、`CONFIG_SPIRAM is not set`（L1054）、`CONFIG_ESP_CONSOLE_UART_NUM=0`（L1147）、`CONFIG_FREERTOS_HZ=100`（L1336）、`CONFIG_TINYUSB_MSC_ENABLED=y`+`CDC_ENABLED=y`（L2123/L2131）、`CONFIG_TINYUSB_TASK_AFFINITY_CPU1`（L2098）。
- **重点**：Flash 实际 2MB（报告/文档写 16MB，**不一致**）；分区表实际用 `partitions_singleapp`（自定义 `partitions.csv` 未启用）。

## 17. `partitions.csv` — 自定义分区表（未生效）

- 内容：`nvs/otadata/phy_init/factory(4MB)/ota_0/ota_1/storage(3.625MB)`（8 行），是 ~16MB OTA 布局。
- **⚠️**：sdkconfig `CONFIG_PARTITION_TABLE_SINGLE_APP=y` → 此 CSV **未启用**（`HANDOVER_AUDIT_STAGE1.md` L26 已指出）。

## 18. `scripts/build_tech_report.py`（1469 行）— 技术报告生成器

- **结构**：`SRC={}` 字典内嵌各模块源码摘录（L16-L924）→ python-docx 辅助函数（L929-L1006）→ `build()` 主流程（L1010-L1469），生成 `ESP_Recorder_技术报告_完整源码解析.docx`。
- **设计原因**：报告与代码同源（代码改了脚本改一处即可重出报告）。

## 19. `scripts/build_report_docx.py`（1212 行）— 完整报告生成器

- **结构**：`CODE_*` 常量内嵌代码片段（L14-L562）→ 辅助函数（L566-L657）→ `build_docx()`（L662-L1212），生成 `ESP_Recorder_项目开发完整报告.docx`（对应 `docs/ESP_Recorder_项目开发完整报告.md`）。
- **设计原因**：同一份报告内容，docx 与 md 双形态交付。

## 20. `docs/ESP_Recorder_项目开发完整报告.md`（817 行）— 需求→交付的"故事"主文档

- **价值**：完整功能清单（16 项 F01-F16，L79-L97）、模块依赖（L152-L170）、数据流（L184-L203）、设计决策表（L799-L811）、已知限制（L731-L739）、待硬件验证清单（L720-L727，注：此为 8.3 早表述，8.4 作者记录已部分实机验证，见 `00_PROJECT_OVERVIEW.md` §六）。
- **面试用法**：所有"需求→设计→交付"问题都从这里取证。

---

## 附：无法走读的文件（空 gitlink，明确告知面试官）

| 目录 | 应有内容 | 状态 |
|---|---|---|
| `firmware/esp_recorder_v1.2_dev/` | usb_cdc_bridge（USB CDC 桥接） | 空 gitlink，无源码 |
| `firmware/esp_recorder_v1.3_release_test/` | cdc 测试协议 / 时间同步 / 安全测试模式 | 空 gitlink，无源码 |
| `app/esp_recorder_app_v1.0/` | Flutter App（lib/ 页面） | 空 gitlink，无源码 |
