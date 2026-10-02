# 《尖塔远征 / Spire-Expedition》技术分析报告

> 本文件是《尖塔远征》的完整技术解剖（约 940 行），作为 ai-passport-game skill 的深度案例参考。主题化提炼见同目录各参考文件；本文件供需要深挖细节（具体代码位置/常量/决策过程）时阅读。

> 分析对象：`D:\aipassport-game\Spire-Expedition-main`（FoloToy AI Passport ESP32-C3 独立游戏固件，v1.4）
> 报告目的：提取一切可复用的嵌入式游戏开发技术，供编写"AI Passport 游戏开发 skill"使用。
> 所有结论均来自对源码的实际阅读，附文件路径与关键常量。日期：2026-10-02。

---

## 目录

1. [项目总览与完整目录结构](#1-项目总览与完整目录结构)
2. [BSP/硬件层实现](#2-bsphardware-层实现)
3. [LVGL 集成与 FreeRTOS 任务结构](#3-lvgl-集成与-freertos-任务结构)
4. [UI/UX 设计（三按键导航 + 页面架构 + 字体方案）](#4-uiux-设计)
5. [游戏架构（数据驱动内容 + 状态机 + 地图/战斗）](#5-游戏架构)
6. [存档系统（V5 页面级存档 + NVS + 版本迁移）](#6-存档系统)
7. [音频方案（IMA ADPCM 音乐 + PCM 音效 + 单任务混音）](#7-音频方案)
8. [启动自检系统（315 项只读检查）](#8-启动自检系统)
9. [资源管线（tools 脚本 + 生成式资产）](#9-资源管线)
10. [构建 / 烧录 / 发布流程](#10-构建--烧录--发布流程)
11. [性能与内存策略](#11-性能与内存策略)
12. [电脑端测试体系（Host Tests + UI 巡检）](#12-电脑端测试体系)
13. [文档工程实践](#13-文档工程实践)
14. [踩坑记录（文档中已明确解决的问题）](#14-踩坑记录)
15. [可复用技术清单（Top 技巧汇总）](#15-可复用技术清单)

---

## 1. 项目总览与完整目录结构

### 1.1 项目定位

- **独立固件**：不基于官方 AI Passport 固件裁剪，而是全新 ESP-IDF 工程，上电直接进游戏。官方仓库只用来核对硬件参数（引脚、SPI 频率、I2C 地址），不复制其 BSP/Recovery/BLE 安装流程/cardid（见 `ARCHITECTURE.md` "关键决定"与 `docs/superpowers/specs/2026-09-03-independent-firmware-design.md`）。
- **技术栈**：ESP-IDF 5.5.4 + LVGL 9.5.0 + esp_lvgl_port 2.9.0 + esp_codec_dev 1.6.2（`firmware/main/idf_component.yml`），目标 esp32c3，Flash 8MB，无 PSRAM。
- **游戏规模**：32 张卡、13 种敌人、6 件遗物、8 个双选项事件、3 职业、3 章 × (5 层双路线 + Boss)；V5 存档；26 音效 + 3 音乐；启动 315 项自检。

### 1.2 目录树（关键文件及职责）

```text
Spire-Expedition-main/
├── README.md                     项目主页：功能、构建、烧录、测试入口
├── ARCHITECTURE.md               模块职责、关键决策、数据流（最重要的架构文档）
├── CONTEXT.md                    开发日志式"当前状态"：每次改动的验证证据（哈希、自检结果）
├── card-game-design.md           玩法与数值设计文档（v1.4）
├── .gitignore                    排除 build/、managed_components/、素材源 zip 等
├── firmware/                     ESP-IDF 工程
│   ├── CMakeLists.txt            project(folotoy_cardgame VERSION 1.4)
│   ├── partitions.csv            游戏专用分区表（见 §10.2）
│   ├── sdkconfig                 完整生成配置（提交进仓库）
│   ├── sdkconfig.defaults        精选关键配置（见 §10.3）
│   ├── dependencies.lock         组件管理器锁文件（LVGL 9.5.0 等精确 hash）
│   ├── main/
│   │   ├── app_main.c            45 行启动序列（见 §1.3）
│   │   ├── CMakeLists.txt        注册组件依赖
│   │   └── idf_component.yml     idf>=5.5.4,<5.6.0；lvgl 9.5.0；esp_lvgl_port 2.9.0；esp_codec_dev 1.6.2
│   ├── components/
│   │   ├── device_display/       ST7789P3 面板 + 背光 + LVGL 接入（device_display.c / panel_st7789p3.c）
│   │   ├── device_input/         ADC 三键采样/消抖/长按（device_input.c / input_decode.c）
│   │   ├── device_i2c/           共享 I2C0 总线（SDA=10, SCL=7）
│   │   ├── device_battery/       CW2017 电量计（520mAh profile）
│   │   ├── device_audio/         ES8311/I2S0 + IMA 解码 + 混音 + 音效调度（device_audio.c / audio_mixer.c）
│   │   ├── game_assets/          生成的 LVGL 图片常量（game_assets.c，39559 行）+ 嵌入音频 + 音频元数据
│   │   ├── game_core/            纯 C 战斗规则，不依赖 ESP-IDF/LVGL（game_core.c）
│   │   ├── game_content/         卡牌/敌人/事件/遗物/路线 只读数据表（game_content.c，1080 行）
│   │   ├── game_progress/        职业定义与解锁规则（game_progress.c）
│   │   ├── game_save/            NVS 读写 + V1–V5 编解码（game_save.c / game_save_codec.c 599 行）
│   │   ├── game_settings/        设置编解码 + NVS（game_settings.c / game_settings_codec.c）
│   │   ├── game_state/           页面状态机（game_state.c 2370 行）+ 电量提醒纯逻辑（battery_warning.c）
│   │   ├── game_self_test/       315 项启动自检（game_self_test.c）
│   │   └── game_app/             LVGL 渲染 + 战斗表现队列 + 动效 + 主循环（game_app.c 2633 行）
│   │       ├── battle_presentation.c  战斗动作分段播放队列
│   │       ├── game_ui_v3.c           18 页布局常量表（Figma 对齐）
│   │       ├── game_ui_font_zh_14.c   14px 中文子集字体（5542 行，生成物）
│   │       └── game_ui_font_title_24.c 24px 标题字体（仅"尖塔远征"4 字）
│   └── tests/host/               电脑端测试（见 §12）
│       ├── CMakeLists.txt        cardgame_host_tests 可执行文件 + 4 个 ctest
│       ├── test_*.c/.py          规则/存档/设置/音频/平衡/UI 测试
│       └── ui_preview/           PC 端 LVGL UI 巡检（渲染 18 页截图 + 布局校验）
├── tools/
│   ├── convert_game_assets.py    图片 → LVGL RGB565/RGB565A8 C 数组
│   ├── convert_audio_assets.py   音频 → 16kHz mono PCM/IMA-ADPCM
│   ├── generate_ui_font.py       扫描源码字符 → lv_font_conv 生成中文字体子集
│   └── create_promo_video.py     用 Pillow+ffmpeg 生成 12s 宣传片
├── assets/                       素材源（ogg/mp3）、许可记录（CC0/OFL）、宣传图
├── docs/
│   ├── README.md                 文档索引（"当前有效文档" vs "历史记录"分层）
│   ├── ui-v3-spec.md             UI v3 规范（Figma 基准 + 交互规则）
│   └── superpowers/              specs/（7 份阶段设计）+ plans/（12 份执行计划）
└── release/
    ├── folotoy_cardgame_v1.1~v1.4_full_8MB.bin   可从 0x0 写入的完整镜像
    ├── SHA256SUMS.txt            四个版本的 SHA-256
    ├── README.md                 镜像使用边界（整片擦除警告）
    └── RELEASE_NOTES_v1.4.md     简短更新说明
```

### 1.3 启动序列（`firmware/main/app_main.c`，全部 45 行）

初始化顺序是**可复用的关键决策**：

```c
void app_main(void)
{
    ESP_ERROR_CHECK(game_save_init());            // 1. NVS（失败即致命）
    game_progress_t progress;
    ESP_ERROR_CHECK(game_save_load_progress(&progress));  // 2. 永久进度（失败即致命）
    game_settings_t settings;
    esp_err_t settings_err = game_settings_load(&settings);
    if (settings_err != ESP_OK) {                  // 3. 设置失败→用默认值，不致命
        settings = game_settings_defaults();
    }
    ESP_ERROR_CHECK(device_display_init());        // 4. 显示+LVGL
    ESP_ERROR_CHECK(device_i2c_init());            // 5. 共享 I2C
    ESP_ERROR_CHECK(device_audio_init());          // 6. 音频（失败即致命）
    esp_err_t battery_err = device_battery_init(); // 7. 电池（失败仅告警，不阻止游戏）
    game_self_test_report_t self_test = game_self_test_run(20260904u);  // 8. 315 项自检
    ...
    ESP_ERROR_CHECK(device_audio_set_volume(settings.volume));
    ESP_ERROR_CHECK(device_audio_play(AUDIO_CONFIRM));  // 9. 开机确认音
    ESP_ERROR_CHECK(game_app_start(&progress, &settings));  // 10. 游戏任务
    ESP_ERROR_CHECK(device_display_set_backlight(settings.brightness)); // 11. 背光最后开
}
```

要点：
- **容错分级**：NVS/存档/显示/音频失败 = 致命；设置失败 = 默认值；电池失败 = 仅 `ESP_LOGW`，游戏照常运行。
- **背光最后开启**（第 44 行在 `game_app_start()` 首屏渲染完成后才设置亮度），避免开机白屏（CONTEXT.md 记载曾删除开机 RGB 测试屏）。
- 自检放在硬件初始化后、游戏任务启动前，种子用日期 `20260904u`。

---

## 2. BSP/硬件层实现

### 2.1 屏幕驱动（`firmware/components/device_display/panel_st7789p3.c`）

**引脚定义**（第 17–23 行）：

```c
#define LCD_HOST SPI2_HOST
#define LCD_GPIO_SCLK GPIO_NUM_8     // 时钟
#define LCD_GPIO_MOSI GPIO_NUM_9     // 数据
#define LCD_GPIO_CS   GPIO_NUM_1     // 片选
#define LCD_GPIO_DC   GPIO_NUM_20    // 数据/命令
#define LCD_GPIO_RST  GPIO_NUM_NC    // 复位悬空（用 esp_lcd 软复位）
#define LCD_GPIO_BL   GPIO_NUM_21    // 背光
#define LCD_H_RES 240
#define LCD_V_RES 320
#define LCD_DRAW_BUFFER_LINES 20     // LVGL 绘制缓冲 20 行
#define LCD_SPI_TRANSFER_LINES 80    // SPI max_transfer_sz 80 行
#define LCD_PIXEL_CLOCK_HZ (80 * 1000 * 1000)   // ★ 80MHz（不是 40MHz！）
```

**初始化流程**（`panel_st7789p3_init()`）：
1. `spi_bus_initialize(SPI2_HOST, ...)`，`max_transfer_sz = 240*80*2 = 38400` 字节，`SPI_DMA_CH_AUTO`。
2. `esp_lcd_new_panel_io_spi()`：`spi_mode=0`，`trans_queue_depth=10`，`lcd_cmd_bits/param_bits=8`。
3. `esp_lcd_new_panel_st7789()`（官方 esp_lcd 组件内置 ST7789 驱动，`bits_per_pixel=16`，`rgb_ele_order=RGB`）。
4. `esp_lcd_panel_reset()` + `esp_lcd_panel_init()`。
5. 追加 **13 条 P3 专属寄存器初始化命令**（`s_init_cmds[]`，第 43–57 行）——这是 ST7789**P3** 变体区别于普通 ST7789 的关键，逐条 `esp_lcd_panel_io_tx_param()` 发送：

```c
{0xB2, {0x05,0x05,0x00,0x33,0x33}, 5, 0},   // Porch setting
{0xB7, {0x35}, 1, 0},                        // Gate control
{0xBB, {0x21}, 1, 0},                        // VCOM setting
{0xC0, {0x2C}, 1, 0},                        // LCM control
{0xC2, {0x01}, 1, 0},                        // VDV/VRH enable
{0xC3, {0x0B}, 1, 0},                        // VRH set
{0xC4, {0x20}, 1, 0},                        // VDV set
{0xC6, {0x0F}, 1, 0},                        // Frame rate
{0xD0, {0xA7, 0xA1}, 2, 0},                  // Power control 1
{0xD0, {0xA4, 0xA1}, 2, 0},
{0xD6, {0xA1}, 1, 0},                        // ?
{0xE0, {...14 字节正极性 Gamma...}, 14, 0},
{0xE1, {...14 字节负极性 Gamma..., }, 14, 10}, // 最后一条延时 10ms
```

6. `esp_lcd_panel_invert_color(panel, true)`（**颜色反转必须开**）+ mirror(0,0) + swap_xy(0) + gap(0,0) + `disp_on_off(true)`。

> **可复用技巧**：把面板初始化命令做成 `struct {cmd, data[14], data_len, delay_ms}` 数组循环发送，任何 LCD 变体只需替换表。

### 2.2 背光（`device_display.c` 第 24–73 行）

- LEDC PWM：`LEDC_LOW_SPEED_MODE` + `LEDC_TIMER_0` + **10-bit 分辨率 + 5kHz**，通道 `LEDC_CHANNEL_0`，GPIO21。
- `device_display_set_backlight(percent)`：`duty = (1023 * percent) / 100`，支持 0–100 调光；游戏内亮度范围限 20–100。
- 初始 `duty=0`（黑屏），等首屏画完再开——配合 `app_main` 的调用顺序。

### 2.3 三按键输入（`device_input.c` + `input_decode.c`）

**硬件模型**：GPIO0（`ADC1_CH0`）上三个按键通过分压电阻共享一个 ADC 通道；不同按键按下产生不同电压，**通过电压区间判键**（`input_decode.c` 全部 18 行）：

```c
input_key_t input_decode_mv(int mv) {
    if (mv < 0)    return INPUT_KEY_NONE;
    if (mv < 150)  return INPUT_KEY_UP;     // ~0mV
    if (mv < 447)  return INPUT_KEY_DOWN;   // ~300mV
    if (mv < 1900) return INPUT_KEY_OK;     // ~1000mV
    return INPUT_KEY_NONE;                  // 松开（3.3V）
}
```

**ADC 配置**（`device_input_init()`）：
- `adc_oneshot` 新 API（ESP-IDF 5.x），`ADC_UNIT_1` / `ADC_CHANNEL_0`，`ADC_ATTEN_DB_12`，`bitwidth = ADC_BITWIDTH_DEFAULT`。
- 先用 `adc_oneshot_io_to_channel(0, ...)` 验证 GPIO0 确实映射到 ADC1_CH0，不是则报错返回——**运行时引脚自检**。
- 校准优先 curve fitting，回退 line fitting，都不可用则 raw×3300/4095 估算（`try_create_adc_calibration()`，编译宏 `ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED` 双保险）。

**消抖与长按状态机**（`input_task()`，常量在第 9–13 行）：

```c
#define INPUT_TASK_STACK_WORDS 2048      // 8KB
#define INPUT_TASK_PRIORITY 5
#define INPUT_SAMPLE_INTERVAL_MS 20      // 20ms 采样
#define INPUT_STABLE_SAMPLES 3           // 连续 3 次相同才认（60ms 消抖）
#define INPUT_LONG_PRESS_MS 800          // 800ms 长按阈值
```

算法（每 20ms 一轮）：
1. 采样 → `input_decode_mv()` 得 candidate_key。
2. candidate 变化则计数清 1；连续 `INPUT_STABLE_SAMPLES` 次且与 stable_key 不同则切换：**释放时**（原键非空且未发过长按事件）发送 `INPUT_EVENT_PRESS`（短按在**松开瞬间**触发，长按在按住 800ms 时立刻触发，两者互斥 `long_sent` 标志）。
3. 按住期间 `xTaskGetTickCount() - press_tick >= 800` 则发 `INPUT_EVENT_LONG` 并置 `long_sent=true`。
4. 事件通过 `xQueueSend(s_event_queue, ..., 0)` 非阻塞入队（队列由 game_app 创建后传入 `device_input_init(queue)`）——**输入任务不做任何处理逻辑**。

> **可复用技巧**：短按在"松开时"确认、长按在"按住中"确认的设计，保证同一按键动作不会同时产生两种事件。

### 2.4 共享 I2C（`device_i2c.c` + `include/device_i2c.h`）

```c
#define DEVICE_I2C_PORT I2C_NUM_0
#define DEVICE_I2C_SDA 10
#define DEVICE_I2C_SCL 7
#define DEVICE_I2C_ES8311_ADDR 0x18   // 音频 codec（7 位地址）
#define DEVICE_I2C_CW2017_ADDR 0x63   // 电量计
```

- 新版 `i2c_master_bus` API，`glitch_ignore_cnt=7`，**内部上拉**（`enable_internal_pullup=true`）。
- 单例模式：`s_bus` 静态指针，重复调用直接返回 OK；ES8311 与 CW2017 各自 `i2c_master_bus_add_device()` 挂同一总线（CW2017 100kHz，ES8311 控制接口 400kHz）。
- **总线只创建一次**是 ARCHITECTURE.md 明示的关键决策。

### 2.5 CW2017 电量计（`device_battery.c`）

- 寄存器：VERSION=0x00, VCELL=0x02, SOC=0x04, CONFIG=0x08, SOC_ALERT=0x0B, PROFILE=0x10（80 字节）。
- **内置 520mAh 电池 profile**（`s_battery_profile[80]`，十六进制数组，注明移植自 FoloToy 官方 Apache-2.0 代码）。
- 初始化流程：读 version → `profile_matches()` 逐字节对比 → 不匹配则 `update_profile()`（进 SLEEP 模式 0xF0 → 写 80 字节 → 回读校验 → 置 UPDATE_FLAG 0x80 → 回 ACTIVE 0x00）→ 匹配则确认 CONFIG 为 ACTIVE。
- 等待就绪：最多 50 次 × 100ms 轮询 `device_battery_soc() >= 0`，超时返回 `ESP_ERR_TIMEOUT`（且把设备从总线移除）。
- 读数 API：
  - `device_battery_soc()`：读 SOC_H 1 字节即百分比（>100 视为失败返回 -1）。
  - `device_battery_mv()`：读 2 字节，`raw = (hi<<8|lo) & 0x3FFF`，`mv = raw * 3125 / 10000`（CW2017 VCELL 单位 312.5µV 的换算）。

### 2.6 ES8311 音频硬件初始化（`device_audio.c`）

**I2S 引脚**（第 23–27 行）：

```c
#define AUDIO_I2S_PORT I2S_NUM_0
#define AUDIO_I2S_MCLK GPIO_NUM_6
#define AUDIO_I2S_BCLK GPIO_NUM_5
#define AUDIO_I2S_WS   GPIO_NUM_3
#define AUDIO_I2S_DOUT GPIO_NUM_2
#define AUDIO_I2S_DIN  GPIO_NUM_4
#define AUDIO_SAMPLE_RATE 16000     // 16kHz 单声道 16bit
#define AUDIO_BOOT_VOLUME 45
```

**I2S 通道**（`init_i2s()`）：`i2s_new_channel` TX-only，`dma_desc_num=6`、`dma_frame_num=240`（=每块 240 帧×2 字节=480B，6 块 DMA 环形缓冲），`auto_clear_after_cb=true`（DMA 写完后自动补零，防啸叫）。标准 Philips 时序，mono slot，ESP32-C3 做 I2S master。

**Codec**（`init_codec()`，走 esp_codec_dev 组件）：
- `audio_codec_new_i2s_data()`（数据面）→ `audio_codec_new_i2c_ctrl()`（控制面，addr = 0x18<<1，挂共享总线，400kHz）→ `audio_codec_new_gpio()`。
- `es8311_codec_new()`：**DAC-only 模式**（`ESP_CODEC_DEV_WORK_MODE_DAC`，不用录音），`use_mclk=true`，`mclk_div=256`，ES8311 作 slave。
- `esp_codec_dev_open(16000/16bit/mono)` + `esp_codec_dev_set_out_vol(45)`。
- sdkconfig 中只保留 `CONFIG_CODEC_ES8311_SUPPORT=y`，其余 12 种 codec 全关（省 flash）。

---

## 3. LVGL 集成与 FreeRTOS 任务结构

### 3.1 移植方式：esp_lvgl_port 组件（不手写 flush_cb/indev）

项目**没有手写** `flush_cb`、`indev_read_cb` 或 tick——全部交给 `espressif/esp_lvgl_port 2.9.0`。`device_display.c` 第 91–118 行：

```c
lvgl_port_cfg_t lvgl_config = ESP_LVGL_PORT_INIT_CONFIG();
lvgl_config.task_stack = DISPLAY_LVGL_TASK_STACK_BYTES;   // 8192 字节
lvgl_port_init(&lvgl_config);

const lvgl_port_display_cfg_t display_config = {
    .io_handle = s_panel_io,
    .panel_handle = s_panel,
    .buffer_size = panel_st7789p3_lvgl_buffer_pixels(),   // 240*20 = 4800 像素
    .double_buffer = false,                               // ★ 单缓冲
    .hres = 240, .vres = 320,
    .color_format = LV_COLOR_FORMAT_RGB565,
    .flags = { .buff_dma = true, .swap_bytes = true },    // DMA 缓冲 + 字节序交换
};
s_display = lvgl_port_add_disp(&display_config);
```

- esp_lvgl_port 内部创建独立任务跑 `lv_timer_handler`，并提供 `lvgl_port_lock(timeout)/unlock()` 互斥锁。
- **`swap_bytes=true`** 解决 SPI 大端 RGB565 字节序问题（SPI 发送是 MSB first，RGB565 在内存是小端）。
- **单缓冲 4800 像素 = 9600 字节 DMA 内存**——比双整屏（150KB×2）省巨量 RAM，是 ESP32-C3 无 PSRAM 下的现实选择。
- LVGL 刷新周期 `CONFIG_LV_DEF_REFR_PERIOD=33`（≈30fps 上限）。
- 输入不走 LVGL indev：三键事件队列 → game_app 任务 → game_state 状态机 → 主动重绘（**拉模式**，不是 LVGL 默认的焦点组模式，理由见 §4.1）。

### 3.2 线程安全约定（ARCHITECTURE.md 数据流 + 代码）

- **任何非 LVGL 线程不得直接调 LVGL**，必须 `device_display_lock(timeout_ms)`（封装 `lvgl_port_lock`）。
- game_app 主循环每次渲染都 `if (device_display_lock(0)) { render...; device_display_unlock(); }`——**锁不到就跳过本帧**，绝不阻塞游戏逻辑。
- 输入/音频对界面只做**非阻塞入队**（`xQueueSend(..., 0)`）。

### 3.3 任务全景表

| 任务 | 文件 | 栈 | 优先级 | 职责 |
|---|---|---|---|---|
| main task | app_main.c | 3584B（CONFIG_ESP_MAIN_TASK_STACK_SIZE） | 默认 | 初始化后直接返回（游戏已移交） |
| lvgl task | esp_lvgl_port 内部 | **8192B**（`DISPLAY_LVGL_TASK_STACK_BYTES`） | 4（组件默认） | lv_timer_handler + SPI flush |
| device_input | device_input.c | 2048 words=8KB | **5** | 20ms ADC 采样→事件队列 |
| device_audio | device_audio.c | 3072 words=12KB | **5** | IMA 解码+混音+I2S 写入（音频优先级高于界面） |
| game_app | game_app.c | 4096 words=16KB | **4** | 游戏主循环：输入→状态机→渲染→存档→音效 |

`CONFIG_FREERTOS_HZ=100`。互斥体：LVGL 锁（esp_lvgl_port）+ `s_effect_queue_lock`（`portMUX_TYPE` 临界区保护音效队列）。

### 3.4 game_app 主循环（`game_app.c` 第 2440–2585 行，核心可复用模式）

```c
while (true) {
    // 动画播放期 40ms 轮询，平时最长等 1s 输入
    TickType_t wait_ticks = pdMS_TO_TICKS(battle_presentation_active(...) ? 40 : 1000);
    BaseType_t received = xQueueReceive(s_input_queue, &event, wait_ticks);

    if (battle_presentation_active(...)) {          // ① 战斗演出队列逐段推进
        ... battle_presentation_tick(&s_battle_presentation, elapsed_ms);
        if (变化 && device_display_lock(0)) { render_battle_hand(...); unlock; }
        if (received == pdTRUE || still_active) continue;   // 演出期间吞掉输入（锁输入）
    }
    if (received != pdTRUE) {                        // ② 空闲：30 秒电池轮询
        if (now - last_battery_poll_us >= 30000000) { ...battery_warning_update...; }
        continue;
    }
    if (s_battery_alert != NONE) { 消除告警重绘; continue; }     // ③ 低电弹窗优先吃掉一次按键
    if (game_persistence_flow_pending(...)) { persist_completed_run(); continue; }  // ④ 结算持久化

    game_state_update_t update = game_state_handle_input(&s_state, event, esp_timer_get_time());
    update_audio_scene();                            // ⑤ 页面→音乐场景
    bool started = battle_presentation_start(...);    // ⑥ 生成战斗演出队列
    play_feedback(update.audio_lead);                // ⑦ 先播动作音
    play_feedback(update.audio);                     //    再播结果音
    if (update.progress_changed) { persist_completed_run(); }   // ⑧ 通关：先存进度再清 run
    if (update.settings_changed) { 存设置/音量/亮度/背光; }
    if (update.needs_render) { save_run_after_update(); }       // ⑨ 页面级自动存档
    if (update.needs_render && device_display_lock(0)) {        // ⑩ 三档渲染范围
        if (scope == GAME_RENDER_BATTLE_HAND) render_battle_hand(...);
        else if (scope == GAME_RENDER_BATTLE_DYNAMIC && render_battle_dynamic()) {}
        else render_app();                           // 整页重建
        if (scope != FULL && 耗时 > 50000us) ESP_LOGW("slow battle partial refresh");
        device_display_unlock();
    }
}
```

---

## 4. UI/UX 设计

### 4.1 三按键导航：不用 LVGL group/focus，自建页面状态机

**关键决策**：LVGL 的 `lv_group`/`lv_indev` 焦点模型为触摸/编码器设计，与"三键 + 22 个页面 + 每页自管理选中索引"的场景不匹配。项目改为：

- `game_state_t` 内每页一个 `selected_*` 字段（`selected_card`、`selected_shop_item`、`selected_pause_item`、`selected_route_column`……见 `game_state.h` 第 126–192 行巨型状态结构体）。
- `game_state_handle_input()` 按 `state->page` switch 分发到 `handle_battle()`、`handle_pause()`、`handle_reward()` 等每页处理器（`game_state.c` 第 2312 行起）。
- **上下键环形滚动**：`selected = (selected + 1) % item_count`（`handle_battle_move()` 第 974–990 行），0 处回绕到末项。
- 选中态由渲染层表达：金色 2px 边框 vs 普通边框 1px（`create_panel(parent, rect, bg, selected)`）。
- **手牌 + "结束回合"虚拟槽**：`item_count = hand_count + 1`，空手按确认也能结束回合（第 974 行）。

**长短按语义**（战斗页，`handle_battle()` 第 1501–1520 行）：

```c
if (event.type == INPUT_EVENT_LONG && event.key == INPUT_KEY_UP)   return enter_pause(...);      // 长按上=暂停
if (event.type == INPUT_EVENT_LONG && event.key == INPUT_KEY_DOWN) { page = APP_PAGE_DECK; }     // 长按下=卡组
if (event.type == INPUT_EVENT_LONG && event.key == INPUT_KEY_OK)    return handle_battle_end_turn(state); // 长按确认=结束回合
```

### 4.2 页面切换架构：22 页枚举 + 三档渲染范围

- `app_page_t` 枚举 22 页（TITLE/SETTINGS/CLASS_SELECT/MAP/BATTLE/REWARD/EVENT/SHOP/DECK/STATUS/CONFIRM_NEW_RUN/TARGET_SELECT/MONSTER_BOOK/REST/HELP/UNLOCK/RESULT/PAUSE/ERROR/CARD_SELECT/EVENT_RESULT/CHAPTER_REWARD）。
- **渲染范围三档**（`game_render_scope_t`）：`GAME_RENDER_FULL`（整页重建）、`GAME_RENDER_BATTLE_DYNAMIC`（血量/意图/状态原位更新）、`GAME_RENDER_BATTLE_HAND`（仅手牌层）。状态机决定档位，渲染层照办——**渲染层不反推数据变化**（ARCHITECTURE.md 明示）。
- 整页重建：`begin_page_with_title_bar()`（game_app.c 第 684 行）先 `lv_obj_clean(screen)` 清空，再画暗色背景 + 地牢底图 + 标题栏（可选），并**重置所有持久控件缓存指针**（`s_battle_hand_layer=NULL` 等 15 个静态指针）。
- **战斗页例外**：`s_battle_hand_layer`/`s_battle_status_layer` 只创建一次（`if (s_battle_hand_layer == NULL || !lv_obj_is_valid(...))`），之后 `render_battle_hand()` 原位更新 3 个卡牌槽的文本/图标/边框/透明度——CONTEXT.md："三张手牌和状态徽章只创建一次……不再销毁任何战斗图层"。
- **数值飘字用固定对象池**：`FLOATING_LABEL_SLOTS=3` 个循环复用的 label（第 103–105 行），杜绝频繁创建销毁。

### 4.3 布局常量表：game_ui_v3.c（Figma 对齐的数据驱动 UI）

`game_ui_v3.c`（361 行）几乎全是 `const` 布局结构体——**每个控件的 x/y/w/h 都是编译期常量**，与 Figma 设计稿一一对应：

```c
const game_ui_v3_battle_layout_t GAME_UI_V3_BATTLE = {
    .enemy_status = {8, 8, 120, 78},    .enemy_sprite = {140, 44, 92, 80},
    .player_status = {116, 104, 116, 80}, .player_sprite = {12, 104, 88, 80},
    .cards = { {28,210,54,60}, {92,206,56,64}, {158,210,54,60} },   // 3 个手牌槽（中张放大突出）
    .dialogue = {12, 196, 216, 74},     .end_turn = {72, 272, 96, 20},
    ...
};
```

地图节点 11 个（5 层×2 列 + Boss）`GAME_UI_V3_MAP_NODES[]`，路线 20 条 2 点线段 `GAME_UI_V3_MAP_ROUTE_LINES[]`（从底 (120,278) 向上汇到 Boss (120,66)）。渲染层 `obj_rect(obj, rect)` 一次设置位置尺寸。

**颜色调色板**集中在 `game_app.c` 第 110–119 行的内联函数：背景 `#0b090e`、标题栏 `#181016`、面板 `#1a1015`、高亮金 `#f4bc33`、边框 `#8e6125`、红 `#d94b3f`、绿 `#4cc06a`、正文 `#f5ddb0`——**黑金像素风全靠这 10 个常量**。

### 4.4 战斗界面布局（v1.4 宝可梦式对角）

- 怪物图右上 (140,44)，怪物血条/意图/状态左上 (8,8)；角色图左下 (12,104)，角色血条/能量右下 (116,104)；手牌横排底部 y≈210；对话框 (12,196,216,74) 演出时覆盖手牌区。
- 状态用短文字"毒3/易2/弱1"（`game_ui_v3_status_short_name()`），敌我各最多 3 枚徽章（`render_battle_statuses()`）。
- 出牌/敌方行动时：手牌隐藏 → 对话框按 `battle_presentation` 队列逐条显示 520/720ms → 结束后卡牌"回场"动画。

### 4.5 LVGL 动效封装（game_app.c 第 121–223 行，全部可复用）

| 函数 | 效果 | 参数 |
|---|---|---|
| `animate_hit(obj, x)` | 受击抖动 | x±5px、55ms、3 次 + playback |
| `animate_lunge(obj, x, dist)` | 前冲 | 140ms 出 + 180ms 回，ease_out |
| `animate_pulse(obj)` | 缩放脉冲 | scale 256→288（LVGL 9 的 256=100%），150/220ms |
| `animate_target_hit(obj,x,color)` | 抖动+闪色 | `image_recolor` 从 OPA_COVER 渐隐到 TRANSP 360ms |
| `animate_selected_card(obj, y)` | 选中上浮 | y-6px、110ms |
| `animate_map_node(obj)` | 呼吸 | OPA 70↔100、420ms、无限循环 |
| `animate_floating_value(...)` | 飘字 | 上移 18px + 延迟后淡出，用对象池 |
| `animate_battle_end_transition()` | 胜/败横幅 | 800ms 上移 + 阴影 + 350ms 后淡出 |

技巧：动画前 `lv_anim_delete(obj, cb)` 防叠加；`lv_obj_is_valid()` 防野指针；completed 回调里 `LV_OBJ_FLAG_HIDDEN` 回收。

### 4.6 字体方案（中文字体子集，最值得学习的部分）

**问题**：240×320 小屏 + 8MB Flash，全量中文字体（GB2312 一级字库 14px 2bpp ≈ 数百 KB）不可接受；且游戏文案会持续新增。

**方案**（`tools/generate_ui_font.py`，49 行）：
1. **正则扫描**三个组件（game_app/game_content/game_state）所有 `.c/.h` 中 `[\u3400-\u9fff]` 的全部 CJK 字符 + 手写标点集 `"%+-/·×—'''""…↑↓▶◀、。《》【】！（），：；？"`。
2. 调 `npx lv_font_conv --size 14 --bpp 2 --format lvgl --font SourceHanSansSC-Normal.otf --range 0x20-0x7E --symbols <全部字符> --lv-font-name game_ui_font_zh_14`。
   - **2bpp（4 级灰度抗锯齿）+ LVGL 压缩字体格式**（配合 `CONFIG_LV_USE_FONT_COMPRESSED=y`），比 1bpp 好看、比 4bpp 省 50%。
   - 字体源文件直接用 LVGL 组件自带的 `managed_components/lvgl__lvgl/scripts/built_in_font/SourceHanSansSC-Normal.otf`（思源黑体，OFL 1.1 许可，不用额外下载）。
3. 标题字体单独生成：24px、只含"尖塔远征"4 字、不含 ASCII（`game_ui_font_title_24.c`，171 行）。
4. 生成后自动替换 include 为 `#include "lvgl.h"` 并统一换行符。

**结果**：`game_ui_font_zh_14.c` 5542 行 ≈ 3145 行字节数据（约 50KB），覆盖约 700 个字符 + 完整 ASCII。字体注释里保留 `--symbols ...` 完整清单（供测试反查）。

**防缺字回归**（`test_rendering_contract.py`）：提取源码全部字符串字面量中的非 ASCII 字符，与字体文件注释里的 symbol 集合做差集，缺任何一个即测试失败——CONTEXT.md 记载此检查源于"二阶段"等新文案显示为方块的真机事故。

**LVGL 侧配置**：`CONFIG_LV_TXT_ENC_UTF8=y`（且显式关闭 ASCII 编码器）、`CONFIG_LV_USE_FONT_COMPRESSED=y`、默认字体用 Montserrat 14（`CONFIG_LV_FONT_DEFAULT_MONTSERRAT_14=y`，避免默认字体链接失败）。

---

## 5. 游戏架构

### 5.1 分层设计（ARCHITECTURE.md + CMakeLists 依赖验证）

```text
纯逻辑层（可在 PC 上测试，0 依赖 ESP-IDF/LVGL）:
  game_core      战斗规则原子操作（game_battle_play_card_effects 等）
  game_content   只读内容表 + 路线生成/校验 + 意图计算
  game_progress  职业定义/解锁
  game_save      blob 编解码（纯函数，NVS 拆到最薄的一层 esp_err 包装）
  game_settings  设置编解码（纯函数）
  game_state     22 页状态机（依赖 device_input.h 的枚举——用 #ifdef CARDGAME_HOST_TEST typedef 桩）
  battery_warning / battle_presentation / game_persistence_flow   纯逻辑小模块

硬件层:
  device_*       输入/显示/I2C/电池/音频

装配层:
  game_app       唯一同时 import LVGL + 设备 + 状态机的组件
  app_main       启动序列
```

`device_input.h` 的跨平台头（第 5–19 行）是典型手法：host 测试时 `typedef int esp_err_t; typedef void* QueueHandle_t;`，设备编译时才 include 真头——让 game_state 既能在设备跑又能进 PC 测试。

### 5.2 数据驱动内容（`game_content.c`）

所有内容为 `static const` 表 + 宏构造器，**放 Flash rodata，0 堆分配**：

```c
// 卡牌：宏收窄 5 种形态（单效果/双效果/带 trait/状态牌）
CARD_ONE(1, "打击", CARD_ATTACK, GAME_CARD_CLASS_SHARED, 1, GAME_EFFECT_DAMAGE, 6, 9)
CARD_TWO_T(9, "充能", CARD_SKILL, GAME_CLASS_WARRIOR, 0, GAME_EFFECT_ENERGY,1,2, GAME_EFFECT_BLOCK,3,5)
CARD_STATUS(40, "伤口", GAME_CARD_TRAIT_NONE)   // 状态牌：UNPLAYABLE
```

- `card_def_t`：id/name/type/class_id/主效果/次效果/upgraded 数值对/traits 位掩码。**每卡最多 2 个顺序效果**（`GAME_CARD_EFFECT_MAX=2`），升级只改数值不改结构——存档因此可以每卡 3 字节（u16 id + u8 upgraded）。
- 敌人 `enemy_def_t`（id 100–130）：max_hp/intent_damage/intent_block/visual 枚举/passive 枚举/passive_name。
- **敌人意图 = 回合查表**：每个敌人一个 `enemy_intent_t pattern[]`（2–3 招循环），`game_content_enemy_intent(id, turn)` 返回 `pattern[(turn-1) % count]`；Boss 版本 `..._intent_for_hp()` 在 `hp*2 <= max_hp` 时切换到 `_phase_two_pattern`（半血二阶段）。
- 意图结构含施加状态、塞状态牌（`status_card_id/amount`，如 Boss 往你弃牌堆塞"伤口"）。
- 事件 `game_event_def_t`（id 200–207）：双选项，每选项带 label/effect/amount/cost/card/relic_id/unavailable_message（**不可用原因文案内建**）。
- 遗物 `relic_def_t`（id 300–305）：6 种效果枚举 + 数值。
- 经济常量：普通/精英/Boss 战 14/24/32 金币；商店价格 `18 + 卡费×6`；删牌 18；休息恢复 22（`MAP_REST_HEAL`）；Boss 后回 25% 最大生命。

### 5.3 战斗核心（`game_core.c`，纯 C 无依赖）

- 上限：卡组 40（`GAME_DECK_MAX`）、手牌 10（`GAME_HAND_MAX`）、敌人 2（`GAME_ENEMY_MAX`，正式内容固定 1 对 1，旧双怪存档载入时归一）。
- 回牌规则：每回合摸 5 张、能量 3（`GAME_TURN_DRAW_COUNT/GAME_TURN_ENERGY`）。
- **RNG：xorshift32**（`x^=x<<13; x^=x>>17; x^=x<<5`，种子 0 归一为 `0x6d2b79f5`）——状态机、路线生成、商店共用此实现（各自一份静态拷贝），种子存档可复现。
- 伤害修正（`modified_damage()`）：`(amount + 力量) × 虚弱(×3/4) × 易伤(×3/2)`，整数近似分数运算避免浮点。
- 灼伤：回合结束手牌中每张灼伤牌 2 点伤害（`burn_count * 2`）。
- 中毒：回合结束扣中毒值然后 -1（敌我同规则）。
- 结束回合带 trait 掩码：`game_battle_end_turn_with_traits(battle, retain_mask, burn_mask)`——保留牌用 bitmask 表示手牌位置。
- **随机状态 `rng_state` 存档**→ 读档后洗牌序列可复现。

### 5.4 战斗状态机扩展（`game_state.c` 的 play_card_with_rules，第 1005 行起）

`game_core` 只做原子操作，职业机制/遗物/敌人被动在 game_state 层**改写效果参数后再提交**：

```c
// 战士处决：伤害 + class_meter×8
// 法师引爆：伤害或格挡 + class_meter×2
// 盗贼连击：本回合第 3 张牌 +1 伤害（turn_flags 防重复）
// 炼毒瓶遗物：POISON 效果 +1
// 九头蛇被动：连续同类牌 -3 伤害；硬壳被动：每回合首次被攻击 +4 格挡 ...
```

`turn_flags` 位定义（第 12–15 行）：`SHELL_TRIGGERED / ZERO_COST_DRAWN / GHOST_DRAINED / ATTACK_PLAYED / ROGUE_COMBO_USED`——**用 5 个 bit 记录回合内一次性事件**，正是 V5 存档新增字段之一。

### 5.5 地图/路线系统

- 结构：`game_route_t { nodes[3][5][2]; bosses[3]; }`，节点 4 字节（type/content_id/next_mask）。
- **生成**（`game_content_generate_route(seed)`）：每章模板 5 层（战斗/战斗、战斗/精英、事件/事件、战斗/精英、商店/休息），事件池按 seed 洗牌，每层随机决定左右是否交换——**用最小随机性保证结构约束**。
- **校验**（`game_content_route_valid()`）：穷举 `2^5=32` 条路径，断言每条恰好 3 场战斗、中间层 next_mask 全为 3、商店不在中间层、末层必为功能节点且有休息——**生成器与校验器分离**，自检和存档载入都复用校验。
- 前进逻辑（`advance_run_node`）：`run_node++`，超过 5 层则进 Boss、Boss 后 `run_act++` 归零。
- 商店/奖励用**种子派生确定性随机**：`seed = route_seed ^ 0x51f15e5d ^ (act<<16) ^ node`（`load_shop_offers()` 第 169 行）——同一次冒险中离开商店再回来，货品不变。

### 5.6 战斗表现队列（`battle_presentation.h/.c`，可复用的"演出脚本"模式）

出牌/敌方行动产生一条**固定长度分段播放队列**（最多 6 项，每项文本缓冲 64 字节）：

```c
#define BATTLE_PRESENTATION_MAX_ITEMS 6
#define BATTLE_PRESENTATION_TEXT_CAPACITY 64
#define BATTLE_PRESENTATION_ACTION_MS 520    // "使用了【打击】" 520ms
#define BATTLE_PRESENTATION_RESULT_MS 720    // "熔岩史莱姆受到9点伤害" 720ms
```

`battle_presentation_start()` 由一次状态更新构建队列：动作项 → 结果项（伤害/盾击/破盾/治疗，文案 snprintf 生成）→ 卡牌每个附加效果一项 → 机制提示项。`battle_presentation_tick(elapsed_ms)` 推进索引并返回是否有变化（game_app 据此重绘）。**播放期间 game_app 丢弃输入**（主循环 continue），实现"播动画时锁按键"。此模块纯 C，PC 可测（`test_battle_presentation.c`）。

### 5.7 结算持久化状态机（`game_persistence_flow.c`）

通关时必须"先存永久进度（progress++，驱动职业解锁）→ 成功后才清除当前冒险 run"，两步一个微型状态机驱动，任一步失败停留在结算页重试（`persist_completed_run()` 循环调 `game_persistence_flow_step()`）。**避免只写一半导致"通关了但 run 还在"**。

---

## 6. 存档系统

### 6.1 NVS 使用方式（`game_save.c`）

- 命名空间 `"cardgame"`，三个 key：`"progress"`（12 字节）、`"run"`（V5 826 字节）、`"settings"`（16 字节，game_settings.c 里同命名空间）。
- 全部走 `nvs_get_blob/nvs_set_blob + nvs_commit`，每次 open→操作→close，不留长命句柄。
- **读不到 key = 首次运行**，返回默认值（不算错误）；**blob 尺寸/魔数/校验和错误 = 显式失败**（`ESP_ERR_INVALID_SIZE` / `ESP_ERR_INVALID_CRC`），**绝不自动擦除 NVS**（ARCHITECTURE.md 明示决策）。
- `nvs_flash_init()` 遇 `NO_FREE_PAGES/NEW_VERSION_FOUND` 直接报错返回（不静默擦除）。

### 6.2 blob 格式（`game_save_codec.c`，599 行，手工定长编码）

**通用头部**：4 字节 magic（进度 `"CGPR"`、冒险 `"CGRN"`、设置 `"CGST"`）+ 1 字节版本 + 数据 + 末尾 4 字节 **FNV-1a 32 校验和**（`hash=2166136261; hash^=b; hash*=16777619`）。

**V5 冒险存档 826 字节布局**（用 `#define` 命名每个偏移，第 18–45 行）：

| 偏移 | 内容 |
|---|---|
| 0–3 | "CGRN" |
| 4 | 版本 5 |
| 5–7 | 职业/章/节点 |
| 8–11 | 金币 u16 + 生命 u16 |
| 12–13 | 卡组数/遗物数 |
| 14 | active 标志（=1） |
| 15–22 | 列位置/选中路线/保存页/前页/选中卡/选中目标/待定卡 |
| 23 | battle_active |
| 24–27 | **route_seed（u32，路线可从种子重建）** |
| 28–31 | max_hp_bonus u16 + chapter_energy_bonus |
| 32–36 | 当前敌人 id/事件 id/事件选项 |
| 38– | 卡组 40×3 字节 |
| 后续 | 遗物 8×2、路线 (3×5×2+3)×4=108 字节、完整战斗现场（双方 hp/block/status、意图、抽/弃/手牌堆、能量、回合数、**rng_state**）、奖励页 3 卡、商店 3 卡+价格+售罄位图+删牌标志、选牌目的、休息选项、章节遗物 3 选 1 |
| V5 尾部 | **class_meter / cards_played_this_turn / previous_card_type / turn_flags / damage_dealt_this_turn + 消耗堆 40×3 字节** |
| 末 4 | checksum |

**版本尺寸**：V1=156 / V2=640 / V3=655 / V4=698 / V5=826（`game_save_codec.h` 第 16–20 行）。**版本升级 = 只在尾部追加**，老偏移永不改动——这是 V1–V4 能原地迁移的根本原因。

### 6.3 V1–V4 → V5 迁移（`game_save_run_decode()` 第 430 行起）

- 靠 `blob_size + blob[4] 版本字节` 双重判定版本，各版本用当年 checksum 偏移校验。
- **共同主体**（V2 起布局一致）：按当前偏移读出主结构。
- **按版本叠加**：V3+ 读 status；V4+ 读页面状态（奖励/商店/休息/选牌，置 `page_state_valid=true`）；V5 读职业资源 + 消耗堆。
- **V1 特殊路径**（`decode_v1_run()`）：老格式没有路线数据，`route_seed=1`、`saved_page=3`（地图页），直接 `game_content_generate_route(1, ...)` 重建路线。
- **解码后整体验证**（`valid_run_save()`，第 143–206 行）：校验每个字段范围（职业<3、卡数 1–40、堆总数≤40）、每张卡的 id 在内容表中存在（`game_content_card()!=NULL`）、路线合法（`game_content_route_valid()`）、页面状态一致性（如 REWARD 页必须 reward_count>0）——**防损坏存档带崩游戏**。
- 载入后容错（`apply_run_save()`）：路线校验失败→按种子重生成；EVENT_RESULT 页→回地图；旧双怪→归一 1 敌。
- **触发时机**：`game_state_handle_input` 返回 `needs_render` 即整页存档（`save_run_after_update()`）——**每次页面切换都落盘**，断电即续。

---

## 7. 音频方案

### 7.1 存储（Flash，EMBED_FILES）

- `game_assets/CMakeLists.txt`：3 条音乐 `.adpcm` + 26 条音效 `.pcm` 全部 `EMBED_FILES`。
- `game_audio_assets.c` 用 `extern const uint8_t _binary_xxx_start[] asm("_binary_...")` 拿到链接符号，懒初始化成 `game_audio_asset_t {data, byte_count, sample_count, loop_sample, encoding}` 数组。
- **编码选择**：
  - 音乐 = **IMA ADPCM**（4:1 压缩）：title 256KB（512,836 样本≈32s）、battle 480KB（960,000 样本=60s）、boss 267KB（534,616 样本≈33s），共约 1MB Flash。
  - 音效 = **裸 PCM16**（短促、随机访问）：每条 2–16KB，共约 250KB。
  - `audio_meta.h` 由转换脚本生成，写死每条音乐的样本数（解码回环用）。
- 注意：`audio_generated/` 里还有一个 480KB 的 `game_music.adpcm` 孤儿文件（未列入 EMBED_FILES，不进固件）。

### 7.2 播放（`device_audio.c` audio_task，单任务软件混音）

**架构**：**没有用** esp_codec_dev 的高层播放 API 做"多路播放"，而是：

1. 单一 `audio_task`，每次生成 160 样本（10ms）混音缓冲 `int16_t buffer[160]`。
2. 每样本：`audio_ima_next()` 流式解码音乐（一个 nibble 一步，状态 `audio_ima_state_t{sample,predictor,step_index}` 仅 7 字节）→ 乘 **淡入淡出增益 music_gain**（0..3200，=200ms `AUDIO_FADE_SAMPLES`）和 **闪避增益 duck_gain** → 与当前音效 PCM 样本 `audio_mix_saturated()` 饱和相加 → `esp_codec_dev_write()` 写 I2S。
3. **场景切换淡出淡入**：`device_audio_set_scene()` 只写 volatile 变量 + 唤醒队列；音频任务发现 `current_scene != s_scene` 则每样本 `music_gain--` 到 0 后切换曲目重新淡入——**切歌无爆音**。
4. **音效闪避**：`audio_duck_gain_step()`——有音效时音乐增益从 1000‰ 每样本 -25‰ 降到 550‰（降 45%），无音效时回升——对话/反馈更清晰。
5. **空闲休眠**：无音效且音乐关闭时 `xQueueReceive(wake_queue, portMAX_DELAY)` 挂起，0 CPU。
6. 混音公式：`mixed = music_sample * music_gain * duck_gain / (3 * 3200 * 1000)`（3 是预留削顶余量）。

### 7.3 音效调度器（`audio_mixer.c`，纯 C 可测）

- **容量 8 的环形队列**（`AUDIO_EFFECT_QUEUE_CAPACITY`），push/pop 在 `portMUX_TYPE` 临界区内。
- **优先级**（`audio_effect_priority()`）：WIN/LOSE/SHIELD_BREAK=3，REWARD/RELIC/UPGRADE/PURCHASE=2，其余=1，导航（MOVE/CONFIRM/BACK/INVALID）=0。
- **合并**：连续 MOVE 只保留队列里一个（push 时查重直接 return true）。
- **挤占**：队列满时新音效优先级 ≥2 可踢掉队首导航音；否则丢弃并 `ESP_LOGW`。
- **单音效通道**：一次只播一条音效（`audio_sfx_can_dequeue(sfx_active) = !sfx_active`），先后顺序天然保持。
- **编译期一致性**：`_Static_assert((AUDIO_EFFECT_COUNT - 1) == GAME_SFX_COUNT)`（device_audio.c 第 37 行）——枚举与素材表数量错位直接编译失败（CONTEXT.md 记载此坑曾真机触发）。

### 7.4 上层音频语义

- 三音乐场景 `audio_scene_t`：TITLE（标题/设置/职业/帮助页）、MAP（其余非战斗页）、BATTLE、BOSS、SILENT；game_app 每次输入后 `update_audio_scene()` 重算。
- **动作音 + 结果音**：`game_state_update_t` 带 `audio_lead` 与 `audio` 两个字段，主循环先后播放——解决"一次更新只能带一个音效"导致出牌听不到命中声的问题。
- 音量走 codec 硬件音量 `esp_codec_dev_set_out_vol(0..100)`，不做软件乘法。

---

## 8. 启动自检系统

### 8.1 实现（`game_self_test.c`，196 行）

`game_self_test_run(seed)` 在**真实硬件上、游戏启动前**执行，全部只操作内存/RAM，不写 NVS。核心是宏：

```c
static void check(report, condition, category) {
    report->checks++;
    if (!condition) { report->failures++; report->failure_mask |= category; }
}
```

**失败不阻断启动**，只打位掩码日志（app_main.c 第 32–40 行：PASS 打 `checks=315`，FAIL 打 `mask=0x%08lx`）。类别位：ROUTE/EVENT/BALANCE/SETTINGS/BATTERY/AUDIO/STATE/SAVE。

### 8.2 315 项构成（可从代码推出）

| 检查组 | 数量 | 内容 |
|---|---|---|
| check_routes | 256 | 128 个种子 × (生成成功 + `route_valid`) |
| check_events | 24 | 8 事件 id 存在 + 每事件 2 选项 label 非空 |
| check_balance | 6 | 金币 14/24/32、商店 3 档且首价≤42、删牌 18 |
| check_settings | 5 | 设置编解码往返（含 magic/校验和路径） |
| check_battery | 6 | 阈值/迟滞序列（19→LOW，18→NONE，9→CRITICAL，8→NONE，25→复位，19→再 LOW） |
| check_audio | 5 | IMA 解码固定字节序列期望值 + 饱和混音上下界 |
| check_state_and_save | ~13 | **用真实状态机走"开始→职业→地图→切路线→进战斗→长按暂停"，然后存档编码→解码→恢复→再进战斗逐字段比对** |
| 合计 | **315** | 与 README/CONTEXT.md 日志一致 |

### 8.3 关键设计

- **大对象从堆分配**：`game_state_t`/`game_run_save_t`/blob 全部 `calloc`（`check_state_and_save` 第 134–138 行）——ARCHITECTURE.md："避免占满 ESP32-C3 主任务栈"（main 栈只有 3584B）。
- **种子外置**（app_main 传日期 `20260904u`），换种子即换一批随机路线检查。
- 战斗比对函数 `battles_equal()` 精确到每个牌堆的每张卡——**存档往返的端到端保障**。

---

## 9. 资源管线

### 9.1 tools 四个脚本

| 脚本 | 作用 | 输入 → 输出 |
|---|---|---|
| `convert_game_assets.py`（366 行） | 图片 → LVGL C 数组 | 35 张 PNG（DCSS CC0 素材 + Figma 导出）→ `game_assets.c`（39559 行） |
| `convert_audio_assets.py`（134 行） | 音频 → 固件格式 | 3 条 ogg/mp3 + Kenney zip 内 26 段 → `.adpcm`/`.pcm` + `audio_meta.h` |
| `generate_ui_font.py`（49 行） | 字体子集生成 | 扫描源码字符 → `game_ui_font_zh_14.c` + `game_ui_font_title_24.c` |
| `create_promo_video.py`（246 行） | 宣传片 | 真机图 + 游戏画面 → 1080×1920 30fps MP4（Pillow+imageio-ffmpeg） |

### 9.2 图片转换细节（convert_game_assets.py）

- 两种输出格式：**背景 tile → `LV_COLOR_FORMAT_RGB565`**（不透明，240×320 地砖拼接 + 暗色叠加 (5,6,10,116)）；**角色/怪物/图标 → `LV_COLOR_FORMAT_RGB565A8`**（RGB565 颜色面 + A8 alpha 面，保留透明，比 ARGB8888 省 1/4）。
- **精灵先按 alpha 边界裁切再缩放**：`source.getchannel("A").getbbox()` 裁掉透明边 → `Image.Resampling.NEAREST`（**像素风必须最近邻**）等比缩放居中到目标框（背景色仅用于 fit 模式的底衬参考）。
- 输出 C 文件格式：`static const uint8_t xxx_data[]` + `const lv_image_dsc_t xxx = {.header={.cf,.w,.h,.stride=w*2}, .data_size, .data}`——直接被 LVGL image widget 引用。
- **许可白名单机制**：脚本内 ASSETS 表即"允许的素材映射"（源文件缺失直接 FileNotFoundError）；`ASSET_CREDITS.md` 声明只有表内文件是合法输入。
- 实际数据量：battle_bg 240×320×2=150KB，88×88 精灵×3B≈23KB/只……合计约 550KB Flash rodata。

### 9.3 音频转换细节（convert_audio_assets.py）

- 全部归一 **16kHz 单声道 i16**（`mono_16k()`：均值降混 + `np.interp` 线性重采样）。
- 音乐裁剪起点按素材 loop 点选（battle 从 7.5s 起、最长 60s；title 40s；boss 40s）。
- **音效响度归一**：目标 RMS=0.20、削顶 0.92（`normalized * (0.20/rms)`）——解决素材忽大忽小（CONTEXT.md 记载的坑）。
- **IMA ADPCM 编码器用纯 Python 重写**（与固件 `audio_ima_next()` 解码器严格互逆，89 项步长表 + 16 项索引表与 C 侧完全一致）。
- 生成 `audio_meta.h`：每条音乐 `#define GAME_MUSIC_xxx_SAMPLE_COUNT N` + `GAME_MUSIC_LOOP_SAMPLE 0`（解码到尾部自然回绕——解码器 `sample >= sample_count` 时归零状态即无缝循环）。

### 9.4 资产进固件的完整链路

```text
PNG/OGG 源（assets/ + 外部 CC0 包，gitignore 掉 zip）
  → python tools/convert_*.py（手动运行，产物提交仓库）
  → firmware/components/game_assets/game_assets.c（图片）+ audio_generated/*.pcm/.adpcm（音频）
  → CMakeLists EMBED_FILES（音频）/ 直接编译（图片 C 文件）
  → 链接进 Factory 分区 rodata，运行期 0 拷贝（指针直指 flash）
```

---

## 10. 构建 / 烧录 / 发布流程

### 10.1 工程结构

- 顶层 `firmware/CMakeLists.txt` 仅 3 行：`cmake_minimum_required(3.16)` + `include($ENV{IDF_PATH}/tools/cmake/project.cmake)` + `project(folotoy_cardgame VERSION 1.4)`（版本号进日志 `App version: 1.4`）。
- 13 个组件各自 `idf_component_register`，依赖显式声明（REQUIRES），main 汇总。
- 依赖由组件管理器按 `idf_component.yml` + `dependencies.lock`（记录精确版本 + component_hash）拉取，`managed_components/` 不进仓库。

### 10.2 分区表（`partitions.csv`，游戏专用最小布局）

```csv
# Name,     Type, SubType,  Offset,   Size
nvs,        data, nvs,      0x9000,  0x6000,     # 24KB 存档
phy_init,   data, phy,      0xF000,  0x1000,
factory,    app,  factory,  0x10000, 0x7F0000,   # ~7.9MB 单应用分区
```

**无 OTA/无 Recovery/无 otadata**——独立游戏固件的极简选择，也是与官方固件的根本区别（spec 文档明示）。0x0–0x9000 留给 bootloader（second stage 在 0x0，分区表在 0x8000）。

### 10.3 sdkconfig 关键项（sdkconfig.defaults 精选 + 生成的 sdkconfig）

```ini
CONFIG_IDF_TARGET="esp32c3"
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y          # 8MB Flash
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y          # DIO + 80MHz
CONFIG_ESPTOOLPY_FLASHFREQ_80M=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y      # 用 USB-Serial-JTAG 控制台（不用 UART）
CONFIG_ESP_CONSOLE_UART_DEFAULT=n
CONFIG_COMPILER_OPTIMIZATION_SIZE=y       # -Os（省 flash 优先）
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_160=y     # C3 满频 160MHz
CONFIG_ESP_MAIN_TASK_STACK_SIZE=3584
CONFIG_FREERTOS_HZ=100
# 无任何 CONFIG_SPIRAM 项（C3 无 PSRAM，也不用 QSPI PSRAM 配置）

# LVGL（CONFIG_LV_CONF_SKIP=y，Kconfig 全权接管 lv_conf.h）
CONFIG_LV_COLOR_DEPTH_16=y                # RGB565
CONFIG_LV_MEM_SIZE_KILOBYTES=64           # LVGL 内部堆 64KB
CONFIG_LV_MEM_POOL_EXPAND_SIZE_KILOBYTES=0
CONFIG_LV_USE_BUILTIN_MALLOC=y            # 用 lv_alloc 内置池而非系统堆
CONFIG_LV_DEF_REFR_PERIOD=33              # ~30fps
CONFIG_LV_OS_NONE=y
CONFIG_LV_DRAW_BUF_STRIDE_ALIGN=1
CONFIG_LV_DRAW_BUF_ALIGN=4
CONFIG_LV_DRAW_LAYER_SIMPLE_BUF_SIZE=24576
CONFIG_LV_USE_DRAW_SW=y                   # 纯软件渲染（C3 无 2D 加速）
CONFIG_LV_DRAW_SW_SUPPORT_RGB565A8=y      # 必开（素材格式）
CONFIG_LV_DRAW_SW_SUPPORT_RGB565_SWAPPED=y
CONFIG_LV_USE_FONT_COMPRESSED=y           # 压缩字体解码
CONFIG_LV_TXT_ENC_UTF8=y                  # UTF-8 中文
# CONFIG_LV_TXT_ENC_ASCII is not set
CONFIG_LV_FONT_MONTSERRAT_14=y            # 默认字体兜底
CONFIG_LV_USE_LABEL/BUTTON/IMAGE/BAR/LINE=y   # 只开用到的 5 种 widget
CONFIG_LV_BUILD_EXAMPLES=n / DEMOS=n

# esp_codec_dev 只开 ES8311，其余 12 种 codec 全 n
CONFIG_CODEC_ES8311_SUPPORT=y
```

> **可复用技巧**：`CONFIG_LV_CONF_SKIP=y + CONFIG_LV_CONF_MINIMAL=y` 让 LVGL 全部经 Kconfig 配置，sdkconfig.defaults 即"LVGL 配置的事实来源"，host 预览另用独立 `lv_conf.h`（tests/host/ui_preview/lv_conf.h）。

### 10.4 日常构建/烧录（README.md）

```powershell
. "C:\esp\v5.5.4\esp-idf\export.ps1"     # 激活 IDF 5.5.4
idf.py build                              # 构建
idf.py -p COM15 flash                     # 增量烧录（不清 NVS）
idf.py -p COM15 monitor
```

### 10.5 完整 8MB 镜像发布（release/）

- 产物 `folotoy_cardgame_v1.4_full_8MB.bin`：**8,388,608 字节整片镜像**，含 bootloader(0x0) + 分区表(0x8000) + app(0x10000) + 空白数据区，可从 0x0 一次写入（CONTEXT.md：三区域与构建产物逐字节比对验证）。仓库未记录具体打包命令（标准做法是 `esptool.py --chip esp32c3 merge_bin`，此处以 CONTEXT.md 的"逐段匹配"验证描述为准）。
- 烧录流程（README）：先 `esptool erase_flash`（清原系统+存档），再 `write_flash 0x0 full.bin`，每步 PowerShell 检查 `$LASTEXITCODE`。
- 校验链：`release/SHA256SUMS.txt` 四版本哈希 + README 内嵌 v1.4 哈希 `7FED946D...F939B73`；CONTEXT.md 还记录增量烧录时 bootloader/分区表/app 三段写入后的回读哈希。
- 镜像体积：app `0x249730`≈2.34MB，Factory 剩余 `0x5a68d0`（71%）。
- **无 CI**（无 .github/）——验证全靠 host ctest + 手动 PowerShell 脚本（每命令 `$LASTEXITCODE` 检查防静默失败）。

### 10.6 版本发布节奏（CONTEXT.md 汇总）

每次发布 = 源码 + 文档同步 + host 测试 4 组 0 失败 + 18 页巡检图 + `idf.py build` + 完整镜像生成与逐段校验 + 真机增量烧录（保留 NVS）+ 启动日志确认（App version / self_test PASS 315 / game_app started）→ 提交 Git。**CONTEXT.md 逐条记录每次的 SHA-256 与验证证据**。

---

## 11. 性能与内存策略

### 11.1 内存预算（ESP32-C3 ~400KB SRAM，无 PSRAM）

| 项 | 大小 | 来源 |
|---|---|---|
| LVGL 绘制缓冲 | 240×20×2 = **9.6KB**（单缓冲，DMA） | panel/LCD_DRAW_BUFFER_LINES |
| LVGL 内部堆 | **64KB**（CONFIG_LV_MEM_SIZE_KILOBYTES） | sdkconfig |
| 音频 DMA | 6×240×2 ≈ 2.9KB + 混音 buffer 320B | device_audio.c |
| 输入/音频/游戏任务栈 | 8+12+16KB | 各任务 |
| LVGL 任务栈 | 8KB（曾 4KB 溢出） | device_display.c |
| game_state_t 等 | 静态 BSS（game_app.c `static game_state_t s_state`） | — |
| 自检临时大对象 | calloc 堆分配用完即 free | game_self_test.c |
| 图片/字体/音频 | **全在 Flash rodata**，运行期指针直读 0 复制 | EMBED/const |

### 11.2 性能技巧汇总

1. **单缓冲 20 行分块渲染**：LVGL 脏区重绘只搬 9.6KB/次，SPI 80MHz 下足够 30fps 交互。
2. **渲染范围三档**（FULL/DYNAMIC/HAND）：切牌只刷手牌层（`GAME_RENDER_BATTLE_HAND`），出牌只原位更新数值控件；主循环对非 FULL 刷新计时，>50ms 打 WARN 日志（性能回归监控）。
3. **持久控件 + 固定对象池**：战斗卡牌槽/状态徽章/飘字 label 创建一次反复改属性，避免 lv_obj 创建销毁与 LVGL 堆碎片。
4. **软件渲染裁剪**：LVGL 只编译 LABEL/BUTTON/IMAGE/BAR/LINE 五种 widget，`LV_DRAW_SW_ASM_NONE`（不用汇编优化，-Os 体积优先）。
5. **音频流式解码**：IMA 状态 7 字节，逐 nibble 解码 1MB 音乐 0 预载；空闲时任务挂起。
6. **内容表 const 化**：32 卡+13 敌+8 事件+6 遗物全在 rodata，查找是 O(n) 线性扫（n≤32 无所谓）。
7. **无动态字符串**：状态机消息用枚举 `game_state_message_t`，渲染层 switch 出 const 字符串；飘字用 `lv_label_set_text_fmt` 复用对象。
8. **编译 -Os + 断言保留**（`CONFIG_COMPILER_OPTIMIZATION_ASSERTIONS_ENABLE=y`，RELEASE 级别但 asserts 开着调试真机问题）。

---

## 12. 电脑端测试体系

### 12.1 结构（firmware/tests/host/）

- **CMake 独立工程**（不依赖 ESP-IDF）：把 8 个纯逻辑组件源码直接编进 `cardgame_host_tests`，设备头用 `#ifdef CARDGAME_HOST_TEST` 桩（esp_err_t/QueueHandle_t typedef）。MSVC 下 `/utf-8`。
- **14 个 C 测试文件**：core/content/progress/save_codec/persistence_flow/battle_presentation/input_decode/state/ui_v3/settings/audio_mixer/self_test + **test_game_balance（2304 局模拟：256 种子 × 3 策略 × 3 职业，跑真实状态机整局，断言首章完成率/通关率/第三章生命中位数/回合数/金币）**。
- **2 个 Python 测试**（ctest 集成）：
  - `test_rendering_contract.py`：字面量字符 vs 字体子集、UTF-8 配置、素材 alpha 格式、预览 CMake 一致性——**静态契约测试**。
  - `test_audio_assets.py`：关键音效 RMS ≥ 0.14、时长上限——**感知响度测试**。
- 4 个 ctest：cardgame_host_tests / rendering_contract / ui_preview_screenshots / audio_asset_perceptibility。

### 12.2 UI 巡检（tests/host/ui_preview/）——最有借鉴价值的部分

**同一份 game_app.c 在 PC 上用 LVGL 渲染并截图校验**：

- CMake 把 LVGL 源码（managed_components）+ 全部游戏源码 + 桩（esp_log/esp_timer/freertos stubs）编成 `cardgame_ui_preview.exe`，定义 `CARDGAME_HOST_TEST=1 CARDGAME_LVGL_PREVIEW=1`。
- `ui_preview_main.c`：`lv_test_display_create(240,320)` + RGB565 → 用 LVGL 自带 lodepng 导出 PNG；循环 18 页（`game_app_preview_render_inspect_page(i)`，状态机内建 `s_ui_inspect_pages[]` 巡检页表）+ 10 步真实流程（`game_app_preview_render_flow_step(i)`）。
- **--strict-layout 布局断言**（全部可复用）：
  - `validate_object_tree`：任何可见对象不得出屏、不得出父容器（大背景图豁免）、label 高度 ≥ 字体行高（防截字）。
  - `validate_dense_page_label_spacing`：任意两 label 矩形不得重叠（飘字层豁免）。
  - `validate_title_bar_policy`：18 页每页"应有/不应有标题栏"的期望表。
  - `validate_image_aspect_ratios`：image 的 scale_x==scale_y（禁止拉伸变形）。
  - `validate_background_...`：全屏背景必须 (0,0) 铺满且不透明（防白闪回归）。
  - 特定页内容断言：第 5 页必须有"精英"标签、第 6 页必须有"熔岩史莱姆"、流程第 8 步"战斗胜利"、盾反馈 label 必须是最前层。
- `run_ui_preview.py` 一键 cmake+build+run+清点（必须恰好 18 张 page_*.png、≥6 张 flow_*.png）。
- **设备端无调试入口**：巡检只存在于 PC（CONTEXT.md 明确删除了真机上的 18 页调试预览）。

---

## 13. 文档工程实践

- **四层文档**（docs/README.md 索引）：
  1. `README.md`——对外（构建/烧录/测试入口）。
  2. `CONTEXT.md`——**倒序开发日志**，每条含：改了什么 + host 测试结果 + 构建产物大小 + 镜像 SHA-256 + 真机烧录串口 + 启动日志证据。冲突时"以此文件和代码为准"。
  3. `ARCHITECTURE.md` / `card-game-design.md` / `docs/ui-v3-spec.md`——当前架构/玩法/UI 事实。
  4. `docs/superpowers/{specs,plans}/`——**历史设计与执行记录**，明确标注"可能包含被真机反馈替代的旧方案，不作为实现依据"。
- **specs（要做什么/为什么）与 plans（怎么做/做到哪）分离**，plans 用 TDD 复选框（"写失败测试→确认失败→实现→预览通过"）。
- **素材许可作为文档**：ASSET_CREDITS.md / AUDIO_CREDITS.md 记录每个文件的作者、许可（CC0/OFL）、SHA-256、下载页——发布合规。
- **验证证据文化**：任何"已完成"结论必须附带测试数量、哈希、日志行（CONTEXT.md 风格），这对 AI 辅助开发尤其有价值（防幻觉）。

---

## 14. 踩坑记录（全部来自 CONTEXT.md / ARCHITECTURE.md 已解决问题）

1. **LVGL 软件绘制栈溢出**：加入大图片后 `lv_draw_sw_blend_image_to_rgb565()` 在 4096 栈下溢出（COM15 日志实证）→ LVGL 任务栈提到 **8192 字节**。（ARCHITECTURE.md）
2. **页面切换白闪/白底**：把 240×320 背景塞进 224×250 负坐标裁剪容器 + 全屏透明动画 → 改为所有页面从 (0,0) 画不透明全屏暗背景，并加"背景不得半透明/不得裁剪"巡检断言。
3. **战斗反馈被面板遮挡**：飘字 label 先创建、后建面板层级更高 → 每次显示反馈时 `lv_obj_move_to_index(label, -1)` 提到最前，并加"盾反馈必须最前层"回归检查。
4. **新文案缺字变方块**："二阶段"等新字符不在生成字体子集里 → 字体自动再生成 + **源码字符 vs 字体符号集合的静态测试**。
5. **怪物名与图不符 / 地图精英节点显示"战斗"**：映射 switch 漏 `GAME_NODE_ELITE` 落入 default → 补齐并加"敌人编号/名称/图片类型逐项回归"。
6. **一次状态更新只有一个音效**：出牌听不到命中声 → `game_state_update_t` 增加 `audio_lead`（先动作音后结果音）；音效时长/响度差异大 → 素材重新归一 RMS=0.20。
7. **音效枚举与素材数量错位**：新音效被驱动 `ESP_ERR_INVALID_ARG` 拒绝 → `_Static_assert(AUDIO_EFFECT_COUNT-1 == GAME_SFX_COUNT)` 编译期拦截。
8. **存档格式/校验错误**：不自动擦除 NVS，显式返回错误码留在原页重试；通关持久化拆成"先 progress 后 clear-run"状态机防半写。
9. **旧版本双怪存档**：载入后归一 `enemy_count=1`、清 target/pending（`apply_run_save` 和 `handle_battle` 双重归一）。
10. **V1 存档无路线数据**：用 route_seed=1 重建路线、落回地图页。
11. **电池初始化失败**：只告警不阻断游戏（真机可能出现电量计不在线）；`battery_warning` 加迟滞（<20% LOW、<10% CRITICAL、≥25% 才重新武装），避免 19%↔20% 抖动刷弹窗。
12. **设置下限**：音量最低 10%、亮度最低 20%，禁止 0（防"黑屏以为坏了"）。
13. **开机白屏**：删除 RGB 测试屏；背光在首屏渲染完成后才开启（app_main 顺序）。
14. **性能回归**：非全页刷新 >50ms 打 WARN（`slow battle partial refresh`），为切牌卡顿提供量化监控。
15. **裁牌性能**：早期每次切牌销毁重建全部卡牌控件 → 改为只创建一次 + 三档渲染范围（CONTEXT.md "切牌性能"条目）。
16. **ADC 引脚自检**：运行时 `adc_oneshot_io_to_channel()` 确认 GPIO0→ADC1_CH0，映射不符直接报错（防错板）。
17. **I2C 总线重复创建**：ES8311 与 CW2017 共总线，device_i2c 做成单例（ARCHITECTURE.md 明示决策）。
18. **DMA 啸叫**：I2S `auto_clear_after_cb=true` 让 DMA 空闲时输出静音。

---

## 15. 可复用技术清单（编写 skill 的素材）

**十条最重要的技术点**：

1. **esp_lvgl_port 移植三板斧**：`lvgl_port_init(task_stack=8192)` + `lvgl_port_add_disp(单缓冲 240×20 DMA, swap_bytes, RGB565)` + `lvgl_port_lock/unlock` 做跨线程互斥；不手写 flush_cb/tick。
2. **ST7789P3 面板代码**：SPI2 主机 @80MHz，引脚 SCLK=8/MOSI=9/CS=1/DC=20/BL=21，13 条 init 命令表 + `invert_color(true)`；背光 LEDC 5kHz 10-bit PWM。
3. **ADC 分压三键**：GPIO0/ADC1_CH0 电压判键（<150mV 上 / <447mV 下 / <1900mV 确认），20ms 采样 ×3 次稳定消抖，800ms 长按；短按松开触发、长按按住触发互斥；事件入 FreeRTOS 队列。
4. **无 indev 的三键 UI 架构**：纯 C 页面状态机（22 页枚举 + 每页 selected 索引 + handle_input switch），状态机返回结构化 update（render_scope 三档 / audio_lead+audio / message / visual / progress_changed / settings_changed），渲染层照单执行。
5. **中文子集字体管线**：正则扫源码收集 CJK 字符 → `lv_font_conv --bpp 2`（压缩）→ 静态测试保证"代码里出现的每个中文字都在字体里"；UTF-8 编码固定。
6. **音频全链路**：音乐 IMA ADPCM 流式解码（4:1）+ 音效 PCM16，单音频任务 10ms 块饱和混音，场景 200ms 淡出淡入，音效 45% 闪避，8 槽优先级队列（合并导航音/高优挤占）；Python 端写互逆编码器与响度归一。
7. **手工定长 blob 存档 + FNV-1a 校验 + 尾部追加式版本演进**：NVS namespace "cardgame" / key progress|run|settings；V1→V5 靠"版本字节+blob 尺寸"判定、老偏移不动、新字段追加；route_seed 存档重建路线；battle rng_state 存档复现洗牌。
8. **315 项启动只读自检**：路线穷举生成校验、内容表完整性、存档编解码往返（真实状态机走流程→存→读→逐字段比对）、阈值/混音单元断言；失败只打位掩码不阻断；大对象 calloc 防栈溢出。
9. **PC 端同源 UI 巡检**：host CMake 把 game_app.c + LVGL + stubs 编成 exe，渲染 18 页 + 10 流程步出 PNG，断言出界/重叠/标题栏策略/图片比例/背景不透明/关键文案存在——嵌入式 UI 的"截图回归测试"。
10. **发布工程**：单 Factory 分区 0x10000/0x7F0000 无 OTA；完整 8MB 镜像从 0x0 烧录（先 erase_flash）；SHA256SUMS + 三段回读校验；增量烧录保留 NVS；CONTEXT.md 记录每次发布的全部验证证据。

**其他值得吸收的细节**：xorshift32 随机（种子 0 归一 0x6d2b79f5）；种子派生商店/奖励（同冒险内容稳定）；生成器/校验器分离的路线系统；演出队列（6 项 × 520/720ms）锁输入播动画；先进度后清档的持久化状态机；`_Static_assert` 锁枚举-资源表一致性；CW2017 80 字节 profile 写入与 520mAh 配置；共享 I2C 单例（ES8311 0x18 @400k / CW2017 0x63 @100k）；素材许可清单化（CC0/OFL + SHA-256）。

---

## 附录 A：关键常量速查表

| 类别 | 常量 | 值 | 文件 |
|---|---|---|---|
| 屏 | SPI 像素时钟 | 80 MHz | panel_st7789p3.c:29 |
| 屏 | LVGL 缓冲 | 240×20 px 单缓冲 DMA | panel_st7789p3.c:27 / device_display.c:102 |
| 屏 | 引脚 | SCLK=8 MOSI=9 CS=1 DC=20 RST=NC BL=21 | panel_st7789p3.c:18-23 |
| 输入 | 判键阈值 | <150/<447/<1900 mV | input_decode.c |
| 输入 | 采样/消抖/长按 | 20ms / 3 样本 / 800ms | device_input.c:11-13 |
| 音频 | I2S 引脚 | MCLK=6 BCLK=5 WS=3 DOUT=2 DIN=4 | device_audio.c:23-27 |
| 音频 | 采样率 | 16000 Hz mono 16bit | device_audio.c:29-31 |
| 音频 | DMA | 6 desc × 240 frame | device_audio.c:66-67 |
| 音频 | 淡入淡出 | 3200 样本 = 200ms；闪避降至 55% | device_audio.c:35 / audio_mixer.c:51-53 |
| I2C | SDA/SCL | GPIO10 / GPIO7 | device_i2c.h |
| I2C | 地址 | ES8311=0x18, CW2017=0x63 | device_i2c.h |
| 任务 | 栈 | input 8KB(p5) audio 12KB(p5) game 16KB(p4) lvgl 8KB | 各组件 |
| LVGL | 内存 | 内部堆 64KB / 刷新周期 33ms / 5 widget | sdkconfig |
| 分区 | factory | 0x10000 / 0x7F0000 | partitions.csv |
| NVS | namespace/key | "cardgame" / progress,run,settings | game_save.c:12-14 |
| 存档 | blob 尺寸 | progress 12B / run V1 156→V5 826B / settings 16B | game_save_codec.h:15-20 |
| 战斗 | 回牌 | 摸 5 / 能量 3 / 虚弱 ×3/4 / 易伤 ×3/2 | game_core.c:6-8 |
| 战斗 | 经济 | 战 14/24/32 金 / 删牌 18 / 休息 22 / Boss 后回 25% | game_content.c / game_state.c:8 |
| 演出 | 队列 | 6 项 × 64 字符 / 520ms + 720ms | battle_presentation.h:12-15 |
| 自检 | 检查数 | 315 项（种子 20260904） | app_main.c:32 |
| 电量 | 告警 | <20% LOW / <10% CRITICAL / ≥25% 重武装 | battery_warning.c |
| 设置 | 默认/下限 | vol=50(10-100) bright=60(20-100) music/sfx=on | game_settings_codec.c:32 |

## 附录 B：建议 skill 直接引用的文件（教学蓝本）

| 教学主题 | 蓝本文件 |
|---|---|
| 面板驱动 | `firmware/components/device_display/panel_st7789p3.c`（152 行，完整可抄） |
| 按键驱动 | `firmware/components/device_input/`（两个文件共 215 行） |
| 音频全套 | `firmware/components/device_audio/`（3 文件共 ~500 行）+ `tools/convert_audio_assets.py` |
| LVGL 接入 | `firmware/components/device_display/device_display.c`（143 行） |
| 游戏主循环 | `firmware/components/game_app/game_app.c` 第 2440–2632 行 |
| 状态机范式 | `firmware/components/game_state/game_state.c`（handle_battle/handle_pause 等） |
| 存档编解码 | `firmware/components/game_save/game_save_codec.c` |
| 字体管线 | `tools/generate_ui_font.py` + `firmware/tests/host/test_rendering_contract.py` |
| 图片管线 | `tools/convert_game_assets.py` |
| UI 巡检 | `firmware/tests/host/ui_preview/`（main + stubs + runner） |
| 平衡模拟 | `firmware/tests/host/test_game_balance.c`（2304 局自动对战） |

（报告完）
