# AI Passport 硬件权威事实与 BSP 驱动

> 事实来源：官方 `components/bsp/include/bsp_pins.h`（单一事实来源）+ `docs/hardware-design/specifications.zh_CN.md` + 三案例实测。冲突时以实机测量为准。

## 1. 整机规格

| 项 | 值 |
|---|---|
| 形态 | 可穿戴电子胸牌，透明外壳，60×95×8.5mm，50g |
| MCU | ESP32-C3FH8X：单核 RISC-V 160MHz，~400KB SRAM，**无 PSRAM** |
| Flash | 8MB 片内 |
| 屏幕 | ST7789**P3** 240×320 TFT，RGB565，4-line SPI |
| 输入 | 上/下/确定三键，共用 GPIO0 ADC 电阻梯 |
| 电源 | 硬件电源键（0.5s 开机/2s 关机）；520mAh 锂电 + CW2017 电量计 |
| 音频 | ES8311 codec + 麦克风 + 扬声器，I2S 全双工 |
| 无线 | 2.4G Wi-Fi b/g/n（STA 扫描可用，不必联网）；BT5 LE |
| NFC | 被动 NTAG213（NDEF 读写） |
| 充电 | USB Type-C 5V；**USB-Serial-JTAG 在 GPIO18/19（C3 原生 USB）** |

可用堆实测（PokeWalk，砍 BLE 后）：**231KB**（RAM 101K + Retention 113K + 10K + RTCRAM 7K）。

## 2. 显示：ST7789P3（所有路线通用）

### 引脚（官方 bsp_pins.h）

```c
#define BSP_LCD_W        240
#define BSP_LCD_H        320
#define BSP_LCD_SPI_HOST SPI2_HOST
#define BSP_LCD_MOSI     9
#define BSP_LCD_SCLK     8
#define BSP_LCD_CS       1
#define BSP_LCD_DC       20
#define BSP_LCD_RST      (-1)   // 复位脚硬接 3.3V，走 SWRESET 软复位
#define BSP_LCD_BL       21     // 背光 LEDC PWM
#define BSP_LCD_PCLK_HZ  (80 * 1000 * 1000)   // 官方 BSP 80MHz；40MHz 也实测可用（保守）
#define BSP_LCD_SPI_MODE 0
#define BSP_LCD_INVERT_COLOR 1  // 出厂即需反色；换屏后若呈负片改 0
```

背光 LEDC：`LEDC_TIMER_0 / LEDC_LOW_SPEED_MODE / 10-bit / 5kHz / CHANNEL_0`，`duty = 1023*percent/100`。**初始 duty=0（黑屏），首屏画完再开**（防开机白屏）。

### P3 专属初始化命令（区别于普通 ST7789 的关键）

普通 `esp_lcd_new_panel_st7789()` 之后**必须**逐条追加（数组循环发送，任何面板变体只需换表）：

```c
typedef struct { uint8_t cmd; uint8_t data[16]; uint8_t len; uint16_t delay_ms; } st_init_cmd_t;
static const st_init_cmd_t ST7789P3_CMDS[] = {
    {0xB2, {0x05,0x05,0x00,0x33,0x33}, 5, 0},   // Porch setting
    {0xB6, {0x00,0x82}, 2, 0},                   // Display Function Control
    {0xC5, {0x22}, 1, 0},                        // VCOM
    {0xB7, {0x35}, 1, 0},
    {0xBB, {0x21}, 1, 0},
    {0xC0, {0x2C}, 1, 0},
    {0xC2, {0x01}, 1, 0},
    {0xC3, {0x0B}, 1, 0},
    {0xC4, {0x20}, 1, 0},
    {0xC6, {0x0F}, 1, 0},                        // Frame rate
    {0xD0, {0xA7,0xA1}, 2, 0},
    {0xD0, {0xA4,0xA1}, 2, 0},
    {0xD6, {0xA1}, 1, 0},
    {0xE0, {0xD0,0x04,0x08,0x0A,0x09,0x05,0x2D,0x43,0x49,0x09,0x16,0x15,0x26,0x2B}, 14, 0},
    {0xE1, {0xD0,0x03,0x09,0x0A,0x0A,0x06,0x2E,0x44,0x40,0x3A,0x15,0x15,0x26,0x2A}, 14, 10},
};
// 之后: esp_lcd_panel_invert_color(panel, true); mirror(0,0); gap(0,0); disp_on_off(true);
```

### SPI 驱动参数

- `spi_bus_initialize`：`max_transfer_sz = 240*80*2 = 38400`（一次能推 80 行），`SPI_DMA_CH_AUTO`。
- `esp_lcd_new_panel_io_spi`：`spi_mode=0`，`trans_queue_depth=10`，`lcd_cmd_bits/param_bits=8`。
- `esp_lcd_panel_dev_config_t`：`bits_per_pixel=16`，`rgb_ele_order=RGB`。

### 已知面板瑕疵

- 左侧轻微鬼影（ST7789P3 列驱动串扰，硬件限制，无法软件修复）。
- 底部有面板重复行 → Doom 用底部 16 行黑条+红字遮挡（既是遮丑也是签名）。

### 字节序（必踩坑）

ST7789 SPI 大端接收 RGB565；ESP32-C3 小端。两条路：LVGL 侧 `flags.swap_bytes=true`；自推屏侧推屏前原地 bswap16（**截图必须取交换之后的字节**，否则观测看不见字节序类故障）。

## 3. 三按键：ADC 电阻梯

### 电路与阈值（官方 bsp_pins.h 注释全文照录级别的事实）

```
3.3V ── 外部上拉 10k ──┬── ADC 节点(GPIO0 / ADC1_CH0)
                       └── 按键 ── 分压电阻 ── GND
  上(UP)  : 0Ω    →   0 mV
  下(DOWN): 1k    → ~300 mV
  确定(OK): 2.2k  → ~595 mV
  松开    : 无通路 → 3300 mV（上拉）
```

判键窗口（边界取相邻档中点）：**UP `<150mV` / DOWN `150~447mV` / OK `447~1900mV` / 松开 `≥1900mV`**。

- **不能用内部上拉**：约 45kΩ 且精度差，三档全挤到 0~154mV 并随温漂重叠。
- 换阻值时进按键调试页实时看 ADC 电压，逐键记读数取中点改阈值。

### 驱动（ESP-IDF 5.x 新 API）

```c
adc_oneshot_new_unit(&{.unit_id=ADC_UNIT_1}, &adc);
adc_oneshot_config_channel(adc, ADC_CHANNEL_0, &{.bitwidth=ADC_BITWIDTH_DEFAULT, .atten=ADC_ATTEN_DB_12});
// 校准：curve fitting 优先（ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED 双保险），回退 line fitting，
// 都不可用则 raw*3300/4095 估算
adc_cali_create_scheme_curve_fitting(&{...}, &cali);
adc_oneshot_read(adc, ADC_CHANNEL_0, &raw);
adc_cali_raw_to_voltage(cali, raw, &mv);   // → input_decode_mv(mv) 查窗口
```

加分项（Spire 做法）：初始化时 `adc_oneshot_io_to_channel(0,…)` 运行时自检 GPIO0 确实映射 ADC1_CH0，防错板。

### 消抖/长按状态机（Spire 参数，可直接抄）

```
20ms 采样周期；连续 3 次相同才认（60ms 消抖）；
短按(CLICK) 在松开瞬间触发；长按(LONG) 在按住 800ms 时立刻触发；
long_sent 标志保证两者互斥（一次按键动作不会同时产生两种事件）。
事件 xQueueSend(queue, &ev, 0) 非阻塞入队 —— 输入任务不做任何处理逻辑。
任务栈 2048 words (8KB)，优先级 5。
```

BSP 事件模型（官方/PokeWalk）：`PRESS / CLICK / DOUBLE / LONG` × 3 键 = 12 种输入；BSP 在长按完成后**抑制随后的 click**。官方 BSP 用 iot_button 组件（`bsp_button_init` + 回调注册）；Spire/PokeWalk 自写状态机（无依赖，见各自 case-study）。

## 4. I2C：ES8311 + CW2017 共享总线

```c
#define BSP_I2C_PORT         I2C_NUM_0
#define BSP_I2C_SDA          10
#define BSP_I2C_SCL          7
#define BSP_I2C_ES8311_ADDR  0x18   // 7位（8位形式 0x30）
#define BSP_I2C_CW2017_ADDR  0x63
```

- **总线只创建一次**（单例）：ES8311（400kHz）与 CW2017（100kHz）各自 `i2c_master_bus_add_device()` 挂同一总线。重复创建是真实踩过的坑。
- 新版 `i2c_master_bus` API：`glitch_ignore_cnt=7`，`enable_internal_pullup=true`。

### CW2017 电量计要点

- 寄存器：VERSION=0x00 / VCELL=0x02 / SOC=0x04 / CONFIG=0x08 / PROFILE=0x10（80 字节）。
- **内置 520mAh profile**（80 字节十六进制数组，来自 FoloToy 官方 Apache-2.0 代码，直接抄 Spire 的 `device_battery.c`）。
- 初始化：读 version → profile 逐字节比对 → 不匹配则写 profile（SLEEP 0xF0 → 写 80B → 回读 → UPDATE_FLAG 0x80 → ACTIVE 0x00）→ 轮询 SOC 就绪（≤50×100ms）。
- `soc()`：读 SOC_H 1 字节即百分比（>100 视为失败返 -1）。`mv()`：`(hi<<8|lo)&0x3FFF * 3125 / 10000`（VCELL 单位 312.5µV）。
- **电池初始化失败只告警不阻断游戏**（电量计可能不在线）。

## 5. 音频硬件：ES8311 + I2S0

```c
#define BSP_I2S_PORT  I2S_NUM_0
#define BSP_I2S_MCLK  6
#define BSP_I2S_BCLK  5
#define BSP_I2S_WS    3
#define BSP_I2S_DOUT  2    // 播放: MCU → codec
#define BSP_I2S_DIN   4    // 录音: codec → MCU
// PA_CTRL 未接 MCU（功放常通）
```

- 采样率 16000Hz（Spire）或 22050Hz（PokeWalk），单声道 16bit。
- I2S 通道：`i2s_new_channel` TX-only，`dma_desc_num=6, dma_frame_num=240`，**`auto_clear_after_cb=true`**（DMA 空闲输出静音，防啸叫）。ESP32-C3 做 I2S master。
- Codec 走 `esp_codec_dev` 组件（锁 `dependencies.lock`）：i2s 数据面 + i2c 控制面（addr=0x18）→ `es8311_codec_new(DAC-only, use_mclk=true, mclk_div=256, ES8311 作 slave)` → `esp_codec_dev_open(16000/16/mono)`。sdkconfig 只开 `CONFIG_CODEC_ES8311_SUPPORT=y`（其余 12 种 codec 全关省 flash）。
- 音量用硬件音量 `esp_codec_dev_set_out_vol(0..100)`，不做软件乘法。
- **开机炸音**：接了功放的设备，`bsp_audio_boot_quiet()` 放 app_main 第一件事（PokeWalk 教训）。
- 首次 `set_format` 会触发 DMA buf free+alloc（960→480）——I2S 需要**连续** DMA 块，`heap_caps_get_largest_free_block` 与总量一起打日志（最大连续块比总量重要）。

## 6. 启动序列模板（容错分级）

Spire 的 45 行 `app_main` 是标准范本（顺序即决策）：

```c
void app_main(void) {
    ESP_ERROR_CHECK(game_save_init());            // 1. NVS（失败=致命）
    game_progress_t progress;
    ESP_ERROR_CHECK(game_save_load_progress(&progress));  // 2. 永久进度（失败=致命）
    game_settings_t settings;
    if (game_settings_load(&settings) != ESP_OK)  // 3. 设置失败→默认值，不致命
        settings = game_settings_defaults();
    ESP_ERROR_CHECK(device_display_init());       // 4. 显示+LVGL
    ESP_ERROR_CHECK(device_i2c_init());           // 5. 共享 I2C
    ESP_ERROR_CHECK(device_audio_init());         // 6. 音频（失败=致命）
    esp_err_t be = device_battery_init();         // 7. 电池（失败仅告警）
    game_self_test_report_t st = game_self_test_run(SEED);  // 8. 启动自检（只读，不写 NVS）
    ESP_ERROR_CHECK(device_audio_set_volume(settings.volume));
    ESP_ERROR_CHECK(device_audio_play(AUDIO_CONFIRM));       // 9. 开机确认音
    ESP_ERROR_CHECK(game_app_start(&progress, &settings));   // 10. 游戏任务
    ESP_ERROR_CHECK(device_display_set_backlight(settings.brightness)); // 11. 背光最后开
}
```

要点：**背光最后开启**（首屏渲染完成后），避免开机白屏。

## 7. 任务/栈/优先级参照表（两个案例的实测安全值）

| 任务 | 栈 | 优先级 | 职责 |
|---|---|---|---|
| main | 3584~8192B | 默认 | 初始化后返回（游戏已移交） |
| lvgl（esp_lvgl_port 内部） | **8192B** | 4 | lv_timer_handler + SPI flush（4K 会溢出，真机实证） |
| input | 8KB | 5 | 20ms ADC 采样→事件队列 |
| audio | 12KB | 5 | 解码+混音+I2S 写入（**音频优先级高于界面**） |
| game_app | 16KB | 4 | 输入→状态机→渲染→存档 |
| world/后台（如有） | 4KB | 4 | Wi-Fi 扫描/感知/养成结算 |

`CONFIG_FREERTOS_HZ=100`（Spire）或 1000（官方/doom）。`CONFIG_ESP_MAIN_TASK_STACK_SIZE` 相应调小。

## 8. 低功耗与电源行为

- 自动息屏后按功能键唤醒（电源管理在官方固件层）。游戏内自管：60s 无操作熄屏、某键长按手动熄屏（PokeWalk：C 长按全局拦截作熄屏，**任何页面都不能用这个手势**）。
- 熄屏期间可降频后台活动（PokeWalk：扫描 30s→60s）。
- 电池预算思维：Wi-Fi 扫描峰值 200~300mA 脉冲 → **屏幕不能常亮、游戏不能实时**（这是玩法形态的根源约束）。
- RTC 时钟源内部 RC ±5% 温漂（实测 +137ppm）→ 跨关机计时不可靠，需 Wi-Fi 校时 + UTC 锚点补偿（PokeWalk V17 方案）。

## 9. 可抄蓝本文件

| 主题 | 蓝本（相对 `D:\aipassport-game\`） |
|---|---|
| 完整面板驱动（152行） | `Spire-Expedition-main/firmware/components/device_display/panel_st7789p3.c` |
| LVGL 接入（143行） | `Spire-Expedition-main/firmware/components/device_display/device_display.c` |
| 三键驱动（215行） | `Spire-Expedition-main/firmware/components/device_input/{device_input,input_decode}.c` |
| 极简版三键+屏（Doom 用，无 LVGL 依赖） | `ai-passport-doom-main/components/bsp_doom/src/{bsp_display,bsp_button}.c` |
| 共享 I2C | `Spire-Expedition-main/firmware/components/device_i2c/device_i2c.c` |
| CW2017 + 520mAh profile | `Spire-Expedition-main/firmware/components/device_battery/device_battery.c` |
| ES8311 音频全套 | `Spire-Expedition-main/firmware/components/device_audio/` |
| 官方 BSP（iot_button 事件模型） | `C:\Users\49707\WorkBuddy\aiot\ai-passport\components\bsp\` |
