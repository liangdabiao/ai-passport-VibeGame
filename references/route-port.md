# 路线 C：现成引擎移植路线（C 引擎 + 平台垫片）

> 蓝本：《AI-Passport Doom》（`D:\aipassport-game\ai-passport-doom-main`），GBADoom（PrBoom 分支）→ ESP32-C3。适用：把 Doom/复古游戏引擎/解释器（DOOM/Quake/GBA 系/模拟器/脚本解释器）搬上设备。深度分析见 `case-study-doom.md`。

## 1. 引擎选型方法论（先做预算分析再选）

**第一步永远是资源预算**（Doom 的做法，可直接套用）：

| 资源 | 总量 | 系统占用 | 引擎可用 |
|---|---|---|---|
| SRAM | ~400KB | ~100-150KB（FreeRTOS+驱动） | **~200-250KB** |
| Flash | 8MB | ~3MB（系统+app） | **~5MB**（存数据文件） |
| CPU | 160MHz 单核 | ~5% | ~152MHz |

**关键约束：无 PSRAM，所有数据必须挤进 ~250KB 可用 RAM 或从 Flash 实时流加载。**

**选型标准（按重要性）**：
1. **RAM 匹配度**：原平台 RAM 需求 ≤ 我们的可用量（GBADoom 需 256KB ≈ 我们 250KB ✅）。
2. **有同类 MCU 移植先例**：Prusa Core One（STM32F427, 256KB RAM）跑过 GBADoom = 可行性实证。找"在更少 RAM 的硬件上跑过"的引擎。
3. **平台层薄**：核心是纯 C、平台钩子集中在少数文件（i_system/i_video/i_input），便于替换。
4. **已带低内存优化**：GBADoom 为 GBA 做过裁剪深度照明/mip-mapping/压缩查找表。
5. **排除项**：硬性依赖 PSRAM（retro-go ❌）、深绑特定 SDK（rp2040-doom ❌）、默认 RAM 4MB+ 且无压缩路径（doomgeneric ❌）。

**反面参考（极小 RAM 先例）**：MG21DOOM 在 108KB RAM 上跑 Doom 的 15 项优化（32→16→8 位类型降级、枚举→宏、纹理路径裁剪、Flash 直读替代 RAM 缓存）——主方案 RAM 不够时的后备技巧库。

## 2. 平台垫片（替换引擎的平台层）

GBADoom 的 ESP32 垫片只有 2 个文件（`components/doom/`）：

### 2.1 时间：`--wrap=clock` 重定向（必踩坑）

GBADoom 的 `I_GetTime()` 用 newlib `clock()`；**ESP-IDF 的 `clock()` 恒返 0**，引擎计时循环死转饿死看门狗。解法（链接期 wrap）：

```c
// i_system_esp32.c —— 必须返回微秒！I_GetTime 用 CLOCKS_PER_SEC/TICRATE≈28571 除它，
// 比微秒粗的值整除后为 0 = 引擎看到时间静止。
clock_t __wrap_clock(void) { return (clock_t)esp_timer_get_time(); }
```

```cmake
# components/doom/CMakeLists.txt
target_link_options(${COMPONENT_LIB} INTERFACE "-Wl,--wrap=clock")
```

### 2.2 显示：调色板转换 + 条带推屏 + 整数缩放

引擎渲出 240×160 8bpp 调色板索引，物理屏 240×320：

```c
static byte s_backbuffer_data[240 * 160];        // 38.4KB 索引缓冲（static .bss）
static unsigned short s_linebuf_data[240 * 16];  // 7.7KB 条带缓冲
static unsigned short s_palette[256];            // 调色板转 RGB565（预字节交换）

void I_SetPallete_e32(const byte *palette) {     // RGB888 → RGB565 + SPI 字节序
    for (int i = 0; i < 256; i++) {
        unsigned r=*palette++, g=*palette++, b=*palette++;
        unsigned short rgb565 = ((r & 0xF8) << 8) | ((g & 0xFC) << 3) | (b >> 3);
        s_palette[i] = bswap16(rgb565);
    }
}

void I_FinishUpdate_e32(...) {                   // 纵向 2x 拉伸逐条推屏
    static unsigned frame_count;
    if ((++frame_count & 0x3) == 0) vTaskDelay(1);   // 每 4 帧让出一次喂看门狗
    for (int y = 0; y < 160; y += 8) {               // 每条 8 源行
        // 每源行展开成 2 个相同显示行（VSCALE=2）
        ...查 s_palette 填 s_linebuf...
        bsp_display_draw_bitmap(0, y * 2, 240, 16, s_linebuf);
    }
    draw_black_bar_with_text();                      // 底部 16 行黑条遮面板重复行
}
```

### 2.3 输入：三键 → 引擎事件映射

三键玩 FPS 的交互设计（值得学）：**UP=前进；DOWN=右转+使用/确认（一个物理键 post 两个引擎键）；OK=开枪**。菜单里 DOWN 确认选择，游戏里 DOWN 转右+开门。用"按键变化沿"post keydown/keyup 事件对。

### 2.4 致命错误：红屏 + 挂起

```c
void I_Error(const char *error, ...) {
    // vsnprintf 到串口 ESP_LOGE
    // 红屏（整屏 0xF800 条带推送）
    while (1) vTaskDelay(portMAX_DELAY);
}
```

## 3. 大数据文件：自定义 Flash 分区 + mmap 零拷贝

```csv
# partitions.csv —— 引擎数据放独立 data 分区
# Name,   Type, SubType, Offset,    Size,     Flags
nvs,      data, nvs,     0x9000,    0x6000,
phy_init, data, phy,     0xf000,    0x1000,
factory,  app,  factory, 0x10000,   0x300000,   # 3MB 应用
wad,      data, 0x40,    0x310000,  0x4F0000,   # 4.9MB 数据分区
```

```c
int doom_wad_init(void) {
    const esp_partition_t *part = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, "wad");
    spi_flash_mmap_handle_t handle;
    const void *mapped = NULL;
    esp_partition_mmap(part, 0, part->size, SPI_FLASH_MMAP_DATA, &mapped, &handle);  // 零拷贝
    if (memcmp(mapped, "IWAD", 4) != 0) return -1;    // 校验 magic
    doom_iwad = mapped; doom_iwad_len = part->size;
    return 0;
}
// 烧录: esptool.py -p COM4 write_flash 0x310000 DOOM1_PROCESSED.WAD
// （或新版本 esptool write_partition -n wad file）
```

数据文件需要预处理时（如 Doom 的 WAD 合并补丁 + 结构转换），写成 `tools/*.py` 管线并文档化"哪个是最终烧录版"。

## 4. CMake 集成外部引擎源码

```cmake
set(GBADOOM_PATH "${CMAKE_CURRENT_LIST_DIR}/../GBADoom")   # 外部克隆的引擎仓库
set(GBADOOM_SRC "${GBADOOM_PATH}/source")
file(GLOB DOOM_ALL_C "${GBADOOM_SRC}/*.c")
# 排除被替换/不需要的平台文件（i_main.c 有 main()；gba 专属；编译期嵌 WAD 的版本）
list(FILTER DOOM_ALL_C EXCLUDE REGEX "i_main\\.c$")
list(FILTER DOOM_ALL_C EXCLUDE REGEX "i_system_gba\\.cpp$")
list(FILTER DOOM_ALL_C EXCLUDE REGEX "i_system_e32\\.cpp$")
list(FILTER DOOM_ALL_C EXCLUDE REGEX "doom_iwad\\.c$")
idf_component_register(
    SRCS ${DOOM_ALL_C} i_system_esp32.c esp32_wad.c
    INCLUDE_DIRS "." "${GBADOOM_PATH}/include"   # 本目录在前：我们的 doom_iwad.h 覆盖生效
    REQUIRES bsp_doom esp_partition spi_flash
    PRIV_REQUIRES esp_timer)
# 老代码告警全关（-Wno-* 一长串）——移植优先跑通
target_link_options(${COMPONENT_LIB} INTERFACE "-Wl,--wrap=clock")
```

## 5. 内存改造（400KB 无 PSRAM 跑 Doom 的实操）

1. **引擎 zone 内存 256KB → 80KB + 128KB 静态 overflow 缓冲区（.bss）**；overflow 从 bump-only 升级为空闲链表回收+分配列表追踪；`Z_FreeTags` 修成同时遍历 overflow 区（用**地址范围检查**识别 overflow 块、保留原始 tag）。
2. **大数组从全局结构体移到 `r_hotpath.iwram.c` 静态数组**（drawsegs/openings/vissprites，省 ~20KB zone）。
3. **帧缓冲从 malloc 改 .bss 静态**（38KB+8KB）。
4. **任务栈 32KB→16KB**（大数组已移出栈）。
5. sdkconfig 极简：关 BT/WiFi 运行时（`CONFIG_BT_ENABLED=n`；wifi 编译进但从不 init）、`-Os`、日志只 ERROR、`HEAP_POISONING_DISABLED`、main 栈 8192。

## 6. 头文件顺序坑

**`doomtype.h` 必须在 FreeRTOS/stdbool.h 之前 include**——FreeRTOS 把 false/true 定义为宏，会破坏 doomtype.h 的 boolean enum。main.c 第一行就是它。

## 7. 帧率预期与策略

- GBADoom 在 ESP32-C3 上 5~15FPS（关卡复杂度相关）。**提前设定预期并写进 README**。
- 优化方向（未做）：关音频、减渲染行数、降低分辨率。
- 音频：RAM 不够同时跑引擎+音频时先禁用（I2S 输出是后续方向）。

## 8. 移植 checklist（按序执行）

1. 预算分析（§1 表格）→ 候选引擎对比矩阵 → 选型结论写文档（`docs/02-engine-selection.md`）。
2. 克隆引擎到同级目录，CMake GLOB+排除集成（§4）。
3. 写平台垫片：时间（--wrap）/显示（调色板+条带）/输入（三键映射）/错误（红屏挂起）（§2）。
4. 数据分区 + mmap + 预处理管线（§3）。
5. 内存改造迭代：zone 缩小/静态化/栈缩减（§5）——每步真机验证（红屏+串口日志定位）。
6. 踩坑全记录进 `docs/issues.md`（已解决/待解决两张表）。
7. 按键映射在真机上试玩调平衡（单轴三键玩转复杂交互是设计问题不是技术问题）。

## 9. 何时不用这条路线

- 游戏逻辑自己从零写 → 路线 A/B（移植的全部价值在"引擎已存在"）。
- 引擎硬性依赖 PSRAM/双核/GPU → 直接放弃，找替代引擎。
- 引擎数据文件 >5MB 且不可裁剪 → 先做数据压缩/裁剪管线评估。
