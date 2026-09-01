# 10 · 面试问答（20~40 题，答案绑定真实代码）

> 难度：★ 基础 / ★★ 进阶 / ★★★ 深入 / ★★★★ 系统设计。所有"代码证据"指 `firmware/esp_recorder_v1.2/main/`（除非另标）。另有一级证据【作者实习记录】= 作者自述 `研发工作全景总结_20260709-0804_尤译庆.md` §2.3。

## ★ 基础（校招必答）

**1. 这个项目是做什么的？**
→ 双路串口 WiFi 数据记录仪：UART×2 采集 → SD 落盘 + Web 实时监控 + USB U 盘。证据：`README.md` L3-L5、`app_main.c` 初始化流程。

**1b. 你在这个项目里具体负责哪些模块？（分工）**
→ 按【作者实习记录】分工：本人=**双通道数据接入、数据路由、SD 卡落盘、文件管理 API、参数持久化**；Web 实时监控主要由同事（蔡茹锦）负责。对应代码：`uart_driver.c`（接入）、`data_router.c`（路由）、`data_recorder.c`（落盘）、`web_server.c` 文件相关 handler（文件 API）、`config_manager.c`（持久化）。

**2. 为什么选 ESP32-S3？**
→ 需求要 WiFi + 串口 + USB U 盘，S3 一颗芯片有 WiFi + 原生 USB OTG（TypeC 直连 D+/D-，不用 CH340）。证据：`docs/ESP32.md` L121。

**3. 为什么用 ESP-IDF 而非 Arduino？**
→ 任务书明文禁止 Arduino（`docs/=任务.txt` L40-L42）；且需精细控制 TinyUSB 描述符、SD 与 USB MSC 块设备互斥、FreeRTOS 优先级，Arduino 抽象层做不到。

**4. 用了几路 UART？各自接什么？**
→ 2 路：CH1=UART0(TX43/RX44)、CH2=UART2(TX17/RX18)；外加调试控制台（配置为 UART0，与 CH1 冲突——见 ★★★）。证据：`uart_driver.h` L16-L22。

**5. SD 卡怎么接的？文件系统是什么？**
→ SDSPI(SPI2)，CS=10/MOSI=38/MISO=40/SCK=39，20MHz；FATFS 挂 `/sdcard`。证据：`sd_card.c` L16-L22、L61-L67。

**6. USB 是哪种设备？VID/PID？**
→ TinyUSB CDC(虚拟串口)+MSC(U盘) 复合设备，VID=0x303A(Espressif)/PID=0x4002。证据：`usb_cdc.c` L30-L57。

**7. Web 页面是怎么放进固件的？**
→ `EMBED_FILES` 把 `index.html/app.css/app.js` 编成 C 数组，`index_handler` 用 `_binary_*_start/end` 发送。证据：`main/CMakeLists.txt` L3、`web_server.c` L85-L93。

## ★★ 进阶

**7b. 这个项目实机验证到什么程度？**
→ **部分实机验证通过**【作者实习记录 8.4】：SD SPI 挂载 ✓、WiFi AP ✓、Web 管理页 ✓、落盘+fsync 实机运行 ✓、USB 热插拔切换 ✓；另 7.30 记录 CH2→CH1 路由 1 字节丢包 0 验证通过。**双路串口端到端验证当时仍在进行中**【待本人确认最终状态】。注意：报告 12.2 的"待硬件验证"、HANDOVER 的"未烧录"是 8.3 及更早表述，被 7.29 烧录、8.4 验证的更晚记录覆盖——回答时用"部分实机验证通过"，别说"从未烧录"。

**7c. 你在开发中修过哪些真实 bug？**
→ 三个（【作者实习记录】7.28）：①尾部缓存未落盘（`data_recorder.c` L554-L576 `recorder_stop` 先 `s_running=false` 再 flush，v1.2 的 flush 不再检查 s_running）；②Web 输入 ID 冲突（前端控件 id 唯一化）；③USB CDC/MSC 描述符冲突（`usb_cdc.c` L30-L41 配置描述符同时含 CDC+MSC）。详见 `08_DEBUG_AND_PROBLEMS.md` 问题 0。

**8. UART 收到数据到落盘 SD 的完整路径？**
→ `uart_rx_task`(读 100ms) → `recorder_write_rx_channel` → `live_push`(实时流) + `recorder_write_raw`(入 16KB save_rb，`give s_data_sem`) → `rec_io` 任务(≥2KB 或 1s 超时) → `recorder_flush_channel_locked` → `fwrite` → `rec_sync` 每秒 `fsync`。证据：`app_main.c` L53-L67、`data_recorder.c` L308-L366。

**9. 为什么不每收一个字节就写 SD？**
→ 每次写 SD 有 FATFS 元数据 + Flash 擦写固定开销；用环形缓冲攒批减少写次数、延长卡寿命。证据：`build_tech_report.py` L1246 设计说明 + `data_recorder.c` L21-L23。

**10. 双条件落盘是哪两个条件？为什么？**
→ ①缓冲 ≥2KB（`SAVE_THRESHOLD_BYTES=2048`）②距上次写 >1s（`SAVE_TIMEOUT_MS=1000`）。数据快时批量写省开销，数据慢时超时兜底防止数据滞留内存掉电丢失。证据：`data_recorder.c` L22-L23、L316-L327。

**11. 记录文件怎么命名？会不会覆盖？**
→ `/sdcard/REC_CH{n}_YYYYMMDD_HHMMSS[_N].bin`，用 `stat()` 判存在，同秒自动 `_1/_2/...` 后缀。证据：`data_recorder.c` L146-L178。

**12. USB 插上电脑时，SD 卡记录和 U 盘怎么不打架？**
→ `tud_mount_cb` → `recorder_stop()`（落盘关文件）+ `sd_card_unmount_fs()`（只卸 FATFS/VFS，保留 `sdmmc_card_t` 给 MSC）；拔出 `tud_umount_cb` → `sd_card_remount_fs()` 重挂 + 可选恢复记录。证据：`app_main.c` L32-L51、`sd_card.c` L123-L167。

**13. 配置存在哪？为什么？**
→ NVS 为主（`recorder_cfg/config_json`），`/sdcard/config.json` 为可读镜像；用户可直接用电脑改镜像，拔卡也能读 NVS 配置。证据：`config_manager.c` L13-L16、L287-L330。

**14. STA 掉线了怎么办？**
→ 事件回调里重连，最多 3 次 × 5s，成功清零；失败后保持 AP+STA 不退出（AP 仍可用）。证据：`wifi_config.c` L44-L58。

**15. 实时数据 Web 端怎么拿？为什么 200ms？**
→ `GET /api/stream?since=N` 增量拉取（只取 seq>since 的条目），前端 200ms 轮询；since 机制避免重复传输。证据：`data_recorder.c` L609-L634、`web_server.c` L518。

## ★★★ 深入

**16. UART 大数据量会不会丢？怎么处理？**
→ 会丢（buffer 满时 `ringbuf_write` 写多少算多少，差额计入 `drop_count`）。缓解：16KB 缓冲吸收突发 + rec_io 独立任务 + drop 显式统计；但无硬件流控（`UART_HW_FLOWCTRL_DISABLE`），读侧无背压。证据：`data_recorder.c` L299-L303、`uart_driver.c` L57。

**17. SD 写慢（背压）怎么设计？**
→ 三层：缓冲吸收（16KB）→ 独立落盘任务（不阻塞收数）→ 超时兜底（rec_sync 每秒 flush）。缺口是"静默丢包"，改进可加水位告警/硬件流控。证据：`data_recorder.c`、`06_RTOS_AND_CONCURRENCY.md` §五。

**18. FreeRTOS 优先级怎么排的？为什么 rec_sync 最低？**
→ uart_rx(5)、rec_io(5)、tinyusb(5)、rec_sync(1)。fsync 是重操作，放最低避免抢 CPU；rec_io 与 uart_rx 同优先级靠信号量让出。证据：`data_recorder.c` L24-L27、`app_main.c` L182、`sdkconfig` L2094。

**19. USB CDC/MSC 复合设备怎么枚举的？**
→ 一个配置描述符内并列 `TUD_CDC_DESCRIPTOR`(控制接口+数据接口) + `TUD_MSC_DESCRIPTOR`，设备描述符 `bDeviceProtocol=MISC_PROTOCOL_IAD` 表示接口关联复合；MSC 侧实现 `tud_msc_read10/write10_cb` 直读 `sdmmc_card_t`。证据：`usb_cdc.c` L30-L57、`usb_msc.c` L109-L161。

**20. 记录期间能同时下载文件吗？为什么？**
→ 不能。`status_handler` 里 `file_api_available = sd.mounted && !recording`（记录期间禁止文件操作，避免写卡冲突）。证据：`web_server.c` L139-L140。

**21. 配置写失败怎么保证一致性？**
→ `save_or_rollback`：先存旧值 → 改内存 → `config_manager_save`（NVS+镜像）→ 失败则回滚旧值。证据：`config_manager.c` L376-L385。

**22. 你项目里有哪些线程安全问题？怎么保护的？**
→ 计数器用 portMUX（`uart_driver.c` L10、`data_router.c` L12），共享缓冲/状态用互斥量（`data_recorder.c` L49/L57）。遗留：`recorder_get_filename` 无锁读（L457）、`s_data_sem` 在释放 s_mutex 后才 give（L304-L305）。

**23. 为什么实时流不用 WebSocket 而用轮询？**
→ 任务书要求了 WebSocket（`docs/=任务.txt` L218），但实际落地是轮询：实现简单、无长连接状态管理，局域网 200ms 延迟够用。这是"任务书 vs 落地"的显性差异，主动讲出权衡是加分项。

**24. 你觉得这个项目最大的技术难点是什么？怎么解决的？**
→ USB CDC/MSC 与 SD 卡所有权互斥（同一张卡不能既被 FATFS 写又被 USB MSC 读）。解法：mount 回调先停记录、再只卸文件系统保留块设备。证据：`sd_card.c` L133-L150 注释"只卸载 FATFS/VFS，保留 sdmmc_card_t 供 USB MSC"。

## ★★★★ 系统设计 / 挑刺

**25. 如果让你重写 UART 采集层，怎么改？**
→ 事件驱动（UART 事件队列）替代轮询；读侧加水位通知 + 硬件流控；双通道独立波特率；CH1 换 UART1 解除与控制台冲突。

**26. 断电瞬间会丢多少数据？怎么评估？**
→ 最坏丢"缓冲里未 fsync 的数据"：环形缓冲(≤16KB/通道) + FATFS 未落盘，时间窗 ≤1s（rec_sync 周期）。要精确回答需实测；仓库无实测数据，只能给机制上限。改进：文件头加校验 + 恢复标记。

**27. 记录文件的 raw 二进制怎么保证完整性/可解析？**
→ 当前无校验、无文件头，靠文件名时间戳区分；改进加"块序号 + CRC + 尾部闭合标记"，异常断电后启动时扫描恢复。

**28. 两个设备之间跑 Modbus 这类协议，你的记录仪会有什么问题？**
→ 纯透传不做协议解析，Modbus 帧可能在任意字节边界被分到不同 live 行（`LIVE_CHUNK_BYTES=48`），但落盘 `.bin` 是连续字节流不受影响；若需按帧解析需加协议层。

**29. 如果要支持远程服务器上传（需求里写了但没实现），你怎么设计？**
→ 需求 `docs/ESP32.md` L54/L64-L74 要"上传服务器/远程监听/远程 OTA"，仓库未实现（报告 L738 自述"无 MQTT/云"）。设计：STA 连网后走 MQTT/TLS 上传文件 + 下发控制指令；OTA 用 esp_https_ota。标【待本人确认】实际进度。

**30. 你的 sdkconfig Flash 是 2MB，但报告写 16MB，怎么解释？**
→ 诚实答：仓库 sdkconfig 是 2MB 调试配置（`CONFIG_ESPTOOLPY_FLASHSIZE="2MB"` L531、`SINGLE_APP` L545），16MB 是目标量产配置（`partitions.csv` 是 16MB OTA 布局但未启用）；文档与配置持续不一致是遗留问题。标【待本人确认】。

**31. 这个仓库的 git 历史只有一个 commit，你怎么看待？**
→ 承认这是"整理归档后单次初始化提交"（commit `43ae21f`），丢失了增量演进史；版本演进只能靠 6 个目录快照重建。改进：按版本打 tag、保留真实 commit 历史。

**32. v1.2_dev 和 v1.3 的源码为什么不在仓库里？**
→ 它们是空 gitlink（`git ls-files -s` mode 160000，`.gitmodules` 缺失，子模块 commit 无法解析），说明这几个子工程是嵌套 git 仓库、随归档丢失了内容。这是仓库完整性问题，面试需如实说明。

**33. 你的 Flutter App 呢？**
→ 仓库里 `app/` 是空 gitlink；`docs/FLUTTER_ENV_SETUP.md` L127 自述"未创建 Flutter 项目，仅完成环境检查"。所以 App 侧我主要产出是环境评估文档。标【待本人确认】。

**34. 你从接手审计报告里学到什么？（高阶，体现反思）**
→ `HANDOVER_AUDIT_STAGE1.md` 教会我：①区分"代码存在/编译通过/硬件验证"三级证据；②注释与配置要一致（UART0 冲突）；③前后端契约要闭环（API 对齐）；④分区/Flash 要统一。这正是专业嵌入式工程师的"证据意识"。

**35. 时间同步协议（v1.3）你打算怎么实现？**
→ 源码不在仓库，我只能讲设计思路：STA 连网后 SNTP 同步 RTC，再用 `gettimeofday` 打准确时间戳；离线用 CDC 下发时间。标【待本人确认】，切勿假装有代码。

**36. 安全测试模式（v1.3）可能是什么？**
→ 推测是"通过 CDC/USB 下发测试指令、设备回传自检结果"的产测协议。仓库无证据，只能讲推测。标【待本人确认】。

**37. 为什么报告的代码行数（app_main 183 行）和实际（201 行）对不上？**
→ 报告/脚本内嵌的是 v1.1 快照的代码摘录，实际 v1.2 又迭代了（容错降级、configure_channel 等）。这说明"报告要与代码同步更新"，也解释了脚本为何要"生成"报告而非手写。
