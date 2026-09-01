# 08 · 排障与真实问题（STAR 格式，从审计报告 + 代码反推）

> 来源优先级：① **【作者实习记录】** `研发工作全景总结_20260709-0804_尤译庆.md` §2.3（作者自述 7.28 修复的三个问题，一级证据）；② `firmware/esp_recorder_v1.2/docs/HANDOVER_AUDIT_STAGE1.md`（接手审计，7 个高风险问题）；③ 我直接读到的代码；④ 版本 diff。凡审计/作者记录未直接证明、仅由代码推演的风险，标注"【典型风险分析，并非仓库能证明的真实 Bug】"。

## 问题 0：作者 7.28 自述修复的三个问题（一级证据，可代码对照）

| 问题 | 作者自述（7.28） | 代码对照 |
|---|---|---|
| ① 尾部缓存未落盘 | "修复尾部缓存未落盘" | `data_recorder.c` L554-L576 `recorder_stop()`：`s_running=false` 后 `recorder_flush_all_locked()` **不检查 s_running**，先 flush 再 close（旧版 flush 函数因 `!s_running` 提前返回） |
| ② Web 输入 ID 冲突 | "修复 Web 输入 ID 冲突" | 前端 `index.html`/`app.js` 中状态/发送等控件需唯一 DOM id（`web_server.c` L776-L797 与前端 fetch 的接口名对齐） |
| ③ USB CDC/MSC 描述符冲突 | "修复 USB CDC/MSC 描述符冲突" | `usb_cdc.c` L30-L41 `desc_configuration[]` 同时含 `TUD_CDC_DESCRIPTOR` + `TUD_MSC_DESCRIPTOR`（`HANDOVER_AUDIT_STAGE1.md` L119 早指出早期版本"配置描述符只声明 MSC 却仍初始化 CDC"） |

> 这三个修复是**版本演进里最硬的证据**：作者自述（做了什么）与仓库代码（怎么做的）能一一对上。

## 问题 1：UART0 与控制台资源冲突（真实，证据确凿）

- **S 情境**：CH1 用户串口用 UART0，调试控制台也配到 UART0。
- **T 任务**：两路用户串口 + 一个调试控制台要能同时工作。
- **A 动作/证据**：
  - `uart_driver.h` L16 `USER_UART1_NUM = UART_NUM_0`（CH1=UART0）；
  - `sdkconfig` L1147 `CONFIG_ESP_CONSOLE_UART_NUM=0`；
  - `console_cmd.c` L13-L14 注释声称控制台在 "UART1 (TX=17, RX=18)"，但 L80 用 `ESP_CONSOLE_DEV_UART_CONFIG_DEFAULT()` 实际读 UART0——注释、sdkconfig、实际三处不一致；
  - `HANDOVER_AUDIT_STAGE1.md` L114 明确列为"高风险问题 1"。
- **R 结果/风险**：`console_start()` 在 CH1 已装 UART0 驱动后再装控制台，`uart_driver_install` 对已安装端口会失败，且多处用 `ESP_ERROR_CHECK` 会直接 abort；即便不 abort，控制台与 CH1 也争用同一硬件。
- **讲法**：这是**接手审计就发现、当前 v1.2 仍未解决**的问题，体现了"注释与实际配置不一致"的典型嵌入式坑。

## 问题 2：停止记录时环形缓冲可能丢数据（已修复，版本演进亮点）

- **S**：早期版本 `recorder_stop` 先置 `s_running=false` 再 flush，而 flush 函数检查 `s_running`。
- **T**：停止记录必须把缓冲剩余数据完整落盘。
- **A/证据**：`HANDOVER_AUDIT_STAGE1.md` L115 记录"`recorder_stop` 先把 `s_running=false`，随后 `recorder_force_sync` 调用的 `recorder_flush_save_rb` 会因 `!s_running` 立即返回，停止时环形缓冲区残留数据可能丢失"。
- **R/修复证据**：v1.2 `data_recorder.c` L554-L576 `recorder_stop()`：`s_running=false` 后调用的 `recorder_flush_all_locked()` **不检查 s_running**，直接 flush 再 close（L566-L571）——bug 已修复。与【作者实习记录】7.28"修复尾部缓存未落盘"对应。
- **讲法**：这是"接手审计发现问题 → 作者 7.28 修复 → v1.2 代码落定"三级闭环的最有力证据。

## 问题 3：前后端 API 不一致，页面按钮是"假闭环"（已修复）

- **S**：早期前端调用 `/api/wifi/connect`、`/api/wifi/disconnect`、`/api/wifi/scan`、`/api/uart/config`，但后端没注册这些路由。
- **T**：网页配网/串口配置按钮要真正生效。
- **A/证据**：`HANDOVER_AUDIT_STAGE1.md` L40/L44 明确"嵌入前端调用不存在的路由"。
- **R/修复证据**：v1.2 `web_server.c` L776-L797 补齐 `/api/wifi/scan`、`/api/wifi/disconnect`、`/api/wifi/status`、`/api/config/uart`、`/api/config/router`、`/api/config/recorder`（22 路由）。
- **讲法**：从"14 个 API"（报告）到"22 个 API"（v1.2 实际）的演进。

## 问题 4：分区/Flash 配置不一致，App 分区逼近上限（部分遗留）

- **S**：项目自带 `partitions.csv` 是 16MB OTA 布局，但 sdkconfig 用 `partitions_singleapp`，Flash 实际 2MB。
- **T**：构建产物要能放进真实 Flash，且留出余量。
- **A/证据**：`sdkconfig` L531（2MB）/L545（SINGLE_APP）；`HANDOVER_AUDIT_STAGE1.md` L26/L118 记录"App 分区只剩 2%"。
- **R/现状**：v1.2 未改分区策略（sdkconfig 仍 SINGLE_APP + 2MB）。文档仍写"16MB/4MB factory"（`docs/...报告.md` L322），**文档与配置持续不一致**。
- **讲法**：诚实说"量产 16MB 板是目标，仓库里是 2MB 调试配置"，标【待本人确认】。

## 问题 5：WiFi 事件回调内阻塞 5 秒（遗留风险）

- **S**：STA 断线重连在事件回调里 sleep。
- **A/证据**：`wifi_config.c` L44-L53 `WIFI_EVENT_STA_DISCONNECTED` 分支里 `vTaskDelay(pdMS_TO_TICKS(5000))`；`HANDOVER_AUDIT_STAGE1.md` L120 "WiFi 默认事件循环中有 5 秒延迟"。
- **R/风险**：阻塞默认事件循环，期间其他 WIFI/IP 事件（如 AP_START、GOT_IP）无法及时处理。
- **讲法**：这是**尚未修复**的并发/阻塞风险（v1.2 仍存在），改进方向是"重连放独立任务/定时器，事件回调只置标志位"。

## 问题 6：USB CDC 收到数据被丢弃（遗留，v1.2_dev 才补桥接）

- **S**：CDC 虚拟串口收到数据，但没有消费方。
- **A/证据**：`app_main.c` L24-L28 `usb_cdc_rx_handler` 空实现（`(void)data; (void)len;`）；`HANDOVER_AUDIT_STAGE1.md` L87 "USB CDC 收到的数据未解析、未转发"。
- **R/演进**：README 称 v1.2_dev 新增 `usb_cdc_bridge`（CDC↔UART 桥接）——**但 v1.2_dev 源码不在仓库**（空 gitlink），无法取证。v1.2 本身仍丢弃 CDC 数据。
- **讲法**：说明"USB 配网/CDC 透传在 v1.2 还是空闭环，v1.2_dev 才做桥接"，标【待本人确认】。

## 问题 7：HTTP handler 直接做 SD 阻塞 I/O（【典型风险分析】）

- **S**：Web 请求处理里直接读写 SD 卡。
- **A/证据**：`web_server.c` `file_download_handler` L465（`fopen/fread` + 分块发）、`config_manager_save`（写 config.json）；`HANDOVER_AUDIT_STAGE1.md` L120 "HTTP 任务直接执行 SD 配置文件读写"。
- **R/风险**：慢 SD / 卡写满时，httpd 任务被阻塞，状态/实时流等所有 Web 请求都变慢；且与 recorder 的 SD 写竞争。
- **标注**：此条是审计报告的定性风险 + 代码可证的行为，具体卡顿是否在真机复现**无实测数据**，标【典型风险分析】。

## 问题 8：recorder 写失败时"丢字节"统计与文件状态（【典型风险分析】）

- **S**：环形缓冲满 / SD 写失败 / 文件打开失败时数据去向。
- **A/证据**：`data_recorder.c` L299-L303（`written < len` 计入 drop_count）、L194-L196（fwrite 失败置 failed）、L288-L297（fopen 失败丢整批）。
- **R/风险**：有 drop_count 统计（可观测），但"丢"是静默的（无告警机制），长时间记录若 drop 持续上升无主动通知。
- **标注**：机制可证，但"真实丢包率"无数据，**严禁编造吞吐/丢包率**。

---

## 排障方法论小结（面试通用模板）

1. 用日志标签定位：`main/sd_card/usb_cdc/wifi_config/web_server/recorder/uart_driver/config/data_router`（`build_report_docx.py` L1095-L1105）。
2. 用 `esp> sd_test` 测 SD 速度排除存储瓶颈（`console_cmd.c`）。
3. 用 `/api/status` 的 `rec_drop_count / ch*_drop_count / route_*_drop_bytes` 观察丢包（`web_server.c` L148-L182）。
4. 烧录前确认 COM 口是 ESP32 真实串口（VID=303A）而非蓝牙虚拟串口（`docs/...报告.md` L699）。
