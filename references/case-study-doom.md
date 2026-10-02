# 《AI-Passport Doom》技术分析报告

> 本文件是《AI-Passport Doom》的技术解剖，作为 ai-passport-game skill 的深度案例参考（移植路线蓝本）。源项目：`D:\aipassport-game\ai-passport-doom-main`（引擎外部依赖 doomhack/GBADoom，需克隆到同级目录）。所有结论来自对源码与文档的实际阅读。

## 1. 项目定位

在 AI Passport（ESP32-C3，~400KB SRAM 无 PSRAM）上运行 DOOM：
- 引擎：GBADoom（PrBoom 分支，为 GBA 16.8MHz/256KB RAM 优化过的纯 C 引擎）。
- 渲染 240×160 8bpp，纵向 2x 拉伸到 240×320；实测 5~15 FPS。
- WAD 数据文件烧到 Flash 独立分区，mmap 零拷贝读取。
- 仓库本身极小：bsp_doom（硬件抽象）+ doom（平台垫片）+ main + tools（WAD 管线）。

```
ai-passport-doom/
├── components/bsp_doom/        # 硬件抽象（无 LVGL 依赖的极简 BSP）
│   ├── include/bsp_doom.h, bsp_pins.h
│   └── src/bsp_display.c (125行), bsp_button.c (60行)
├── components/doom/            # 引擎封装（平台垫片）
│   ├── i_system_esp32.c        # 时钟重定向/帧缓冲/调色板/推屏/输入/错误
│   ├── esp32_wad.c             # WAD 分区 mmap 加载
│   └── doom_iwad.h             # 覆盖引擎的 IWAD 符号声明
├── main/main.c                 # 80 行入口
├── tools/merge_pwad.py 等      # WAD 预处理管线
├── partitions.csv / sdkconfig.defaults / CMakeLists.txt
└── docs/ (01~04 + WAD-FILES + issues)
```

## 2. 引擎选型方法论（docs/02-engine-selection.md，可直接套用的模板）

**第一步：资源预算表**

| 资源 | 总量 | 系统占用 | 引擎可用 |
|---|---|---|---|
| SRAM | ~400KB | ~100-150KB（FreeRTOS+驱动） | ~200-250KB |
| Flash | 8MB | ~3MB | ~5MB（存 WAD） |
| CPU | 160MHz 单核 | ~5% | ~152MHz |

**第二步：候选对比矩阵（6 方案全评）**

| 方案 | RAM 需求 | 结论 |
|---|---|---|
| GBADoom（doomhack） | ~256KB | ⭐ 选用：RAM 匹配 + Prusa STM32 移植先例 + 纯 C + 自带低内存优化 |
| rp2040-doom | 264KB | 深绑 pico-sdk + 依赖双核 + 超频，工作量 4-6 周 |
| Doomgeneric | 默认 4MB | 接口最简（5 函数）但 RAM 远超预算，无压缩路径 |
| MG21DOOM | 108KB | 优化技巧参考库（类型降级/裁剪/Flash 直读），代码太激进 |
| retro-go / PrBoom | 4-8MB 或需 PSRAM | 硬性排除 |

**选型关键论据**：GBADoom 在 GBA（256KB RAM）上原生运行，且有 2026 年 Prusa Core One（STM32F427, 256KB RAM, ~7FPS）移植先例——**"在同等或更少 RAM 的 MCU 上跑过"是最强的可行性证据**。

## 3. 平台垫片（i_system_esp32.c，529 行，移植的核心工作）

### 3.1 时钟（第一个必踩坑）

GBADoom 的 `I_GetTime()` 用 newlib `clock()`；**ESP-IDF 的 `clock()` 恒返 0** → 引擎计时循环（TryRunTics/D_Wipe）死转 → 饿死 IDLE 任务 → 看门狗。且必须返回**微秒**（I_GetTime 用 `CLOCKS_PER_SEC/TICRATE≈28571` 除它，比微秒粗的值整除后为 0 = 引擎看到时间静止）：

```c
clock_t __wrap_clock(void) { return (clock_t)esp_timer_get_time(); }
// CMakeLists: target_link_options(${COMPONENT_LIB} INTERFACE "-Wl,--wrap=clock")
```

### 3.2 帧缓冲与推屏

```c
static byte s_backbuffer_data[240 * 160];        // 38.4KB 8bpp 索引（static .bss）
static unsigned short s_linebuf_data[240 * 16];  // 7.7KB 条带缓冲
static unsigned short s_palette[256];            // RGB565 预字节交换
#define VSCALE 2        // 160 游戏行 → 320 显示行（纵向 2x 填满全屏）
#define STRIP_H 8       // 每条 8 源行
#define DISP_BOTTOM_BLACK 16   // 底部 16 行黑条遮面板重复行
```

- `I_SetPallete_e32`：RGB888 → RGB565（`((r&0xF8)<<8)|((g&0xFC)<<3)|(b>>3)`）+ `bswap16`（SPI 大端）。
- `I_FinishUpdate_e32`：每 8 源行一条带，每源行复制成 2 个相同显示行；**每 4 帧 `vTaskDelay(1)` 喂看门狗**（引擎平跑不让出）。
- 单缓冲模式：front == back（引擎拿到 backbuffer 指针直接画）。

### 3.3 底部黑条 + 品牌文字

面板底部有重复行瑕疵 → 16 行黑条遮挡 + 红色 "AI-PASSPORT-DOOM" 字样。**教训**：5x7 点阵字体表尝试各种位序全乱码（字体存储格式与渲染逻辑不匹配且难定位），最终**硬编码每个字符的像素坐标**直接写 linebuf——彻底规避字体格式问题。"绕过比排查更快"的务实选择。

### 3.4 输入映射（单轴三键玩 FPS）

```
UP   → KEYD_UP            前进
DOWN → KEYD_RIGHT + KEYD_A 右转 + 使用/确认/菜单进入（一物理键 post 两引擎键）
OK   → KEYD_B             开枪
```

- 按键变化沿 post keydown/keyup 事件对（`s_last_btn` 跟踪；DOWN 的 KEYD_A 用 `down_a_held` 单独跟踪）。
- 菜单里 DOWN 确认选择；游戏里 DOWN 转右+开门（BT_USE 靠近门触发）。
- 无长按无模式切换（早期设计有"长按 OK 切转向模式"，最终简化掉）。

### 3.5 错误处理

`I_Error`：vsnprintf 到 ESP_LOGE → 整屏红(0xF800)条带推送 → `while(1) vTaskDelay(portMAX_DELAY)`。红屏=致命错误的可视化惯例。

## 4. WAD 管线与分区

### 4.1 为什么不能直接烧原版 WAD

1. 需合并 gbadoom.wad 补丁（GBA 专属资源如 `STGANUM0`），否则 `W_GetNumForName: STGANUM0 not found`。
2. seg_t 结构 12→28 字节格式转换，否则 `P_GroupLines: Subsector a part of no sector!`。

```
DOOM1.WAD（官方 Shareware ~4.0MB）
  → python tools/merge_pwad.py DOOM1.WAD gbadoom.wad DOOM1_GBA.WAD   # 合并补丁+大写化 PNAMES
  → GbaWadUtil.exe -in DOOM1_GBA.WAD -out DOOM1_PROCESSED.WAD        # seg_t 转换
  → 烧 DOOM1_PROCESSED.WAD（~3.7MB）到 0x310000
```

### 4.2 分区表与零拷贝加载

```csv
nvs,      data, nvs,     0x9000,   0x6000,
phy_init, data, phy,     0xf000,   0x1000,
factory,  app,  factory, 0x10000,  0x300000,   # 应用 3MB
wad,      data, 0x40,    0x310000, 0x4F0000,   # WAD 4.9MB
```

```c
const esp_partition_t *part = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, "wad");
esp_partition_mmap(part, 0, part->size, SPI_FLASH_MMAP_DATA, &mapped, &handle);  // 零拷贝
if (memcmp(mapped, "IWAD", 4) != 0) return -1;   // 校验 magic
doom_iwad = mapped; doom_iwad_len = part->size;  // 提供与 doom_iwad.c 相同的符号
```

烧录：`esptool.py -p COM4 write_flash 0x310000 DOOM1_PROCESSED.WAD`（首次一次即可）。WAD 二进制不进 git。

## 5. CMake 集成（外部引擎源码）

- `GBADOOM_PATH = ${CMAKE_CURRENT_LIST_DIR}/../GBADoom`（同级目录克隆，路径不同需改）。
- `file(GLOB)` 收集引擎全部 .c，`list(FILTER EXCLUDE)` 排除被替换的：`i_main.c`（有 main()）、`i_system_gba.cpp`、`i_system_e32.cpp`、`doom_iwad.c`（编译期嵌 WAD 版本）。
- INCLUDE_DIRS 本目录在前 → 自带的 `doom_iwad.h` 覆盖引擎头（符号替换生效的关键）。
- 老代码告警全关（一长串 `-Wno-*`）——移植先跑通再治理。
- `SDKCONFIG_DEFAULTS` 变量强制 defaults 生效。

## 6. 内存改造实录（docs/issues.md，每条都是真机红屏换来的）

1. **Z_Malloc 耗尽**：原始 zone 256KB 太大 → 缩到 80KB + 128KB 静态 overflow 缓冲（.bss）；overflow 升级为空闲链表回收+分配列表追踪；修 `Z_FreeTags` 同时遍历 overflow（用**地址范围检查**识别 overflow 块、保留原始 tag）；demo 循环结束时主动 `Z_FreeTags`。
2. **globals_t 大数组挤占 zone**：drawsegs/openings/vissprites 移到 `r_hotpath.iwram.c` 静态数组（省 ~20KB zone）。
3. **帧缓冲分配失败**：帧缓冲 38KB+行缓冲 8KB 从 malloc 改 .bss 静态；任务栈 32KB→16KB。
4. **WiFi 配置无法持久关闭**：构建系统每次把 `CONFIG_ESP_WIFI_ENABLED` 重置回 y → 接受"编译进固件但不调用 esp_wifi_init"，运行时不占额外内存。
5. **demo 模式无法退出**：`G_Responder()` 在 demoplayback 期间 return false 忽略所有键 → 增加 demoplayback 的按键处理（任意 keydown 结束回放进菜单）。

## 7. 头文件顺序与任务结构

- **`doomtype.h` 必须在 FreeRTOS/stdbool.h 之前**：FreeRTOS 把 false/true 定义为宏，破坏 doomtype.h 的 boolean enum（main.c 第一行带注释）。
- 引擎跑在独立 FreeRTOS 任务：栈 16384、优先级 5、`xTaskCreatePinnedToCore(..., 0)`（C3 单核也显式指定）；任务内依次 `I_PreInitGraphics → I_Init → Z_Init → InitGlobals → D_DoomMain`（不返回）。
- sdkconfig 极简：`BT_ENABLED=n`、`COMPILER_OPTIMIZATION_SIZE`、日志只 ERROR、`HEAP_POISONING_DISABLED`、main 栈 8192。

## 8. 已知限制（如实声明）

- 左侧轻微鬼影：ST7789P3 列驱动串扰，硬件限制。
- 声音禁用：RAM 不够同时跑引擎+音频（I2S 是后续方向）。
- 帧率优化未做（可尝试关音频/减渲染行数）。

## 9. 移植路线图（从本项目泛化）

1. 预算分析表 → 候选引擎对比矩阵（RAM 匹配度/MCU 先例/平台层厚度/自带优化）→ 选型结论文档化。
2. 外部引擎同级克隆，CMake GLOB+排除集成，include 优先级做符号覆盖。
3. 平台垫片四件套：时间（--wrap=clock 微秒）/显示（调色板+条带+缩放）/输入（变化沿+多键映射）/错误（红屏挂起）。
4. 大数据文件 → 独立 data 分区 + mmap + 预处理管线（Python）+ "烧哪个"文档。
5. 内存改造迭代（zone 缩小/静态化/栈缩减），每步真机验证。
6. issues.md 两张表（已解决/待解决）全程记录。
7. 按键映射真机试玩调平衡。
