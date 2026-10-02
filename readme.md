---
name: ai-passport-game
description: 为 FoloToy AI Passport 电子胸牌（ESP32-C3、240×320 彩屏、三按键、无 PSRAM）开发游戏固件的完整方法论：硬件约束分析、三条技术路线（LVGL 组件 / 纯 C 自研渲染 / 现成引擎移植）、三键交互设计、中文像素字库、NVS 存档版本化、音频合成与 ADPCM、资产管线、宿主验证门禁、构建烧录与社区发布。只要用户提到 AI Passport、AI 胸牌、电子胸牌、FoloToy Passport、folotoy 游戏机、ESP32-C3 240×320 三键设备，或想在该设备上制作、移植、调试、发布任何游戏或玩法——即使用户只说"帮我做个游戏"——就必须使用本 skill。
---

# AI Passport 游戏开发

FoloToy AI Passport 是一块 60×95mm 的可穿戴电子胸牌：**ESP32-C3（单核 RISC-V 160MHz，~400KB SRAM，无 PSRAM）+ 8MB Flash + ST7789P3 240×320 RGB565 SPI 屏 + 三枚 ADC 分压按键 + ES8311 音频 + CW2017 电量计 + 520mAh 电池**。本 skill 是在该设备上做出高质量游戏的完整方法论，提炼自三个已发布的优秀案例 + 官方仓库工程规范。

## 本机资源（开工前先确认）

| 资源 | 路径 | 用途 |
|---|---|---|
| 案例集合 | `D:\aipassport-game\` | 三个完整游戏源码（见下方案例表），可抄蓝本 |
| 官方仓库克隆 | `C:\Users\49707\WorkBuddy\aiot\ai-passport` | 官方 BSP + AGENTS.md + demo 分支 + 官方 skills |
| ESP-IDF | `D:\esp\esp-idf-v5.5.3` | 工具链（官方基线 5.5.3；Spire 用 5.5.4 亦可） |
| 官方开源仓 | https://github.com/FoloToy/ai-passport | 上游（BSP/文档/skills） |
| 浏览器模拟器 | https://github.com/VOID001/FoloToy-Passport-Simulator | 无真机时的近似验证手段 |

硬件事实冲突时的优先级（官方规定）：**产品规格/实机测量 > `components/bsp/include/bsp_pins.h` > BSP 头文件与实现 > 硬件指南 > README/demo**。这些来源都没有的硬件细节，问用户，不要猜。

## 黄金法则（每个案例用真实事故换来的）

1. **硬件约束先于玩法设计**。520mAh 电池+扫描脉冲电流 → 屏幕不能常亮 → 短会话形态；三按键 → 不能做需要方向键/连点的玩法。最常见的失败方式是"策划做出很好的游戏，然后发现电池撑不住、三个键点不动、flash 塞不下"。动手前先读 `references/game-design-constraints.md` 写设计宪法。
2. **纯逻辑层 0 平台 include**。游戏规则（状态机/数值/判定）放纯 C 模块，只 include `stdint/stdbool`，不用 ESP-IDF/LVGL 任何头——这样它能在 PC 上编译、测试、与参考实现对账。平台层只做搬运与调度。
3. **能在宿主验证的，绝不留到设备上**。设备一轮几十秒且观测弱；PC 上毫秒级、工具全。宿主门禁 + Web/PC 预览是核心开发效率设施，不是锦上添花。
4. **能在编译期挡住的，绝不留到运行期**。布局不变量、存档偏移、枚举-资源表数量一致性，全部 `_Static_assert`。"注释是知识，不是守护"。
5. **内存纪律**：整屏 RGB565 是 150KB，而可用堆实测只有 ~231KB——**永远不要申请整屏帧缓冲**，用横带/分条渲染；大数组必须 static（不入栈、不挤堆）；图片/字体/音频全部放 Flash rodata（EMBED_FILES / const 数组），运行期指针直读零拷贝。
6. **LVGL 非线程安全**：非 LVGL 任务碰 `lv_*` 对象必须持 `bsp_lvgl_lock()`；按键回调只派发轻量事件，音频/存储/网络等慢操作放工作任务。
7. **统一手势语义**：全游戏一套三键语义（上/下=移动，OK=确认，某键长按=返回），提示行如实标注。"UI 承诺的与实际行为不符"是最高优先级缺陷。
8. **中文字体是构建产物**：扫源码字符 → 生成子集 → 缺字测试门禁。UTF-8 正确、编译成功都不代表能显示中文。
9. **观测手段必须能看见目标故障**（问自己：观测点在故障点之后吗？采样率够吗？它坏了我会知道吗？）。截图证明不了动画时序，欠载计数证明不了"有声音"。
10. **编译通过 ≠ 硬件验证**。交付时分开报告 Build / Host tests / Device tests / Unverified。

## 开发工作流

用户说"做一个 XX 游戏"时，按此流程走。每步标注了要读的参考文件（都在本 skill 的 `references/` 下）。

### 第 0 步：确认上下文

- 确认目标设备确实是 AI Passport（问一句，或看用户是否已有该设备工程）。
- 确认起点：全新独立固件？基于官方仓库 fork？还是已有工程续作？
- 确认 ESP-IDF 环境（本机 `D:\esp\esp-idf-v5.5.3`，`export.sh`/`export.ps1` 激活）。缺工具链时读 `references/build-flash-release.md` 的环境节。
- 若在官方仓库克隆里工作，先读它的 `AGENTS.zh_CN.md`（本机路径见上表），遵循其分支/验证规则；官方仓库自带 5 个 skills（passport-develop/setup/build/device-test/debug），已装则用。

### 第 1 步：写设计宪法（先于任何代码）

读 `references/game-design-constraints.md`。产出一份 `docs/01-constitution.md`（或等价物），内容：硬件约束清单 → 每条约束推出的设计决策 → 玩法核心循环 → 页面清单与三键语义表 → 数据规模预算（存档字节数、资产 KB 数、RAM 占用上限）。**这份文档是后续一切争论的裁判。**

### 第 2 步：选技术路线

| 路线 | 适用 | 代表案例 | 深读参考 |
|---|---|---|---|
| **A. LVGL 组件**（esp_lvgl_port + LVGL 9.5） | UI 密集型：卡牌/文字冒险/菜单重的游戏；想要控件/动画/字体生态 | 《尖塔远征》 | `references/route-lvgl.md` |
| **B. 纯 C 自研渲染**（横带软渲染 + 自建页面表） | 像素风、精灵图、追求极致内存/帧率控制、大量自定义绘制 | 《PokeWalk》 | `references/route-native-c.md` |
| **C. 现成引擎移植**（C 引擎 + 平台垫片） | 把 Doom/模拟器/解释器等既有 C 引擎搬上设备 | 《AI-Passport Doom》 | `references/route-port.md` |

**速判**：菜单和文字为主 → A；逐像素画面/复古像素风 → B；引擎已在别处存在 → C。混合也常见（PokeWalk 用官方 BSP 但自建渲染）。选定后读对应 route 文件再动手。

### 第 3 步：搭工程骨架

- 硬件层照 `references/hardware.md`（含全部引脚表、ST7789P3 初始化命令序列、ADC 判键阈值、ES8311/CW2017 要点、启动序列模板、任务/栈/优先级参照表）。三个案例的驱动文件都是可直接抄的蓝本。
- 工程结构（独立固件标准布局）：

```
<game>/
├── firmware/               # ESP-IDF 工程
│   ├── CMakeLists.txt      # cmake_minimum_required(3.16) + include(project.cmake) + project(<name> VERSION x.y)
│   ├── partitions.csv      # 见 references/build-flash-release.md
│   ├── sdkconfig.defaults  # 见 references/build-flash-release.md（有注释的精选配置）
│   ├── main/               # app_main + 页面/应用逻辑
│   └── components/
│       ├── device_*/       # 硬件层（display/input/i2c/audio/battery）
│       └── game_*/         # 游戏层（core 纯逻辑 / content 数据表 / save / state / app 装配）
├── tools/                  # Python 资产管线 + verify_* 门禁（见 references/assets-pipeline.md、testing-quality.md）
├── assets/                 # 素材源 + 溯源/许可记录
├── docs/                   # 宪法/架构/系统设计/CONTEXT 开发日志
└── release/                # 完整 8MB 镜像 + SHA256SUMS
```

### 第 4 步：实现玩法

按主题读参考（均按需、不必全读）：

| 主题 | 参考 | 一句话要点 |
|---|---|---|
| 三键交互与页面导航 | `references/ux-three-keys.md` | 页面表 + enter/exit/key 三函数；长按与短按互斥；重操作配重手势 |
| 中文字库 | `references/fonts-chinese.md` | 扫源码字符→lv_font_conv/自定 1bpp 格式；缺字=构建失败 |
| 存档与状态压缩 | `references/save-system.md` | NVS 整块 blob + magic/版本/校验和 + 前缀保持式升级 + 事务性写入 |
| 音频 | `references/audio.md` | 方案1 运行时合成（零 PCM）；方案2 音乐 IMA-ADPCM+音效 PCM16 单任务混音 |
| 图片/数据资产管线 | `references/assets-pipeline.md` | 上游真源→Python 转换→自描述二进制/C 数组→EMBED_FILES |
| 验证与质量 | `references/testing-quality.md` | 宿主门禁骨架、反向验证三律、观测五问、启动自检、UI 巡检 |

### 第 5 步：验证

1. 纯逻辑宿主门禁（`cc` 编译真 .c + 行协议对账）。
2. UI 在 PC/Web 上预览（路线 A 用 LVGL host 巡检截图；路线 B 用同源预览）。
3. `idf.py build` 通过。
4. 真机烧录验收（**烧录前必须征得用户同意**；设备连接不构成授权）。
5. 交付报告四行格式：

```text
Build:        PASS / FAIL / NOT RUN
Host tests:   PASS / FAIL / NOT RUN
Device tests: PASS / FAIL / NOT RUN
Unverified:   <仍需真机/仪器/用户确认的事项清单>
```

### 第 6 步：发布（用户要求时）

读 `references/build-flash-release.md` 的发布节：merge_bin 完整 8MB 镜像 + SHA256 + 3:4 封面 + 双语介绍 + 社区工作台流程。

## 内存预算速查（刻在脑子里）

| 项 | 数值 |
|---|---|
| SRAM 总量 | ~400KB（无 PSRAM，这是全部） |
| 可用堆（砍 BLE 后实测） | ~231KB |
| 整屏 RGB565 帧缓冲 | 150KB（**装不下，别想**） |
| 横带 240×80 帧缓冲 | 37.5KB（安全值） |
| LVGL 单缓冲 240×20 绘制区 | 9.6KB（路线 A 标配） |
| LVGL 内部堆 | 64KB（`CONFIG_LV_MEM_SIZE_KILOBYTES=64`） |
| 主任务栈 | 3584~8192B（大数组 static，别压栈） |
| 典型分区 | nvs@0x9000(24K) + factory@0x10000(3~7.9MB) |
| 7 个资产 .bin | 158.8KB（PokeWalk 全部精灵+字库） |
| 中文字库 727 字形 16×16 1bpp | 42.4KB |

## 三个案例（蓝本索引）

| 案例 | 路径 | 是什么 | 最值得抄的文件 |
|---|---|---|---|
| 尖塔远征 | `D:\aipassport-game\Spire-Expedition-main` | LVGL 卡牌 Roguelike，32 卡/13 敌/3 职业，V5 页面级存档，315 项自检 | `firmware/components/device_display/panel_st7789p3.c`（152行面板驱动）、`device_input/`（215行三键）、`game_save/game_save_codec.c`（存档编解码）、`tools/generate_ui_font.py`、`tests/host/ui_preview/`（UI 巡检） |
| PokeWalk | `D:\aipassport-game\ESP32-PokemonGo-main` | 纯 C 宝可梦 RPG，Wi-Fi 感知探索，V17 存档，135 个 verify 门禁，Web 同源预览 | `firmware/main/nav.c/h`（页面表）、`screen.c/h`（横带渲染）、`save.h`（前缀保持版本链）、`tools/inspector/`（同源预览）、`docs/10-framework.md`（可迁移方法论） |
| AI-Passport Doom | `D:\aipassport-game\ai-passport-doom-main` | GBADoom 移植，WAD 烧 Flash 分区 mmap 零拷贝，5~15FPS | `components/doom/i_system_esp32.c`（平台垫片：clock 重定向/条带推屏/调色板）、`esp32_wad.c`（分区 mmap）、`docs/02-engine-selection.md`（选型方法论）、`docs/issues.md`（踩坑实录） |

深度技术解剖见 `references/case-study-spire.md`、`references/case-study-pokewalk.md`、`references/case-study-doom.md`（各 250~1050 行，含全部常量与代码位置；只在需要深挖某案例时读）。

## 常见坑（高频复发，先背下来）

- **UART0 控制台与背光冲突**（TX=GPIO21=背光脚）→ 控制台必须用 USB-Serial-JTAG（`CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y`）。
- **ST7789P3 必须反色**（`esp_lcd_panel_invert_color(true)`，出厂即需）+ 13 条 P3 专属 init 命令（普通 ST7789 驱动不带）。
- **SPI 推屏字节序**：RGB565 小端内存 → SPI 大端，必须交换（LVGL 用 `swap_bytes=true`；自推屏手动 bswap）。
- **ADC 按键不能用内部上拉**（45kΩ 精度差，三档挤到 0~154mV 重叠）；判键阈值 0/150/447/1900mV。
- **`esp_lcd_panel_draw_bitmap` 是异步的**：同一缓冲连续推多条带会 DMA 竞争（紫/绿花屏）。同步点方案见 `references/route-native-c.md` §DMA。
- **newlib `clock()` 在 ESP-IDF 恒返 0**：移植老引擎时用 `--wrap=clock` 重定向到 `esp_timer_get_time()`（微秒）。
- **doomtype.h 必须在 FreeRTOS 之前 include**（true/false 宏冲突）。
- **NVS 写入必须 `nvs_commit`**：set 只写缓存，返回成功≠已落盘。
- **Wi-Fi 扫描不联网也能用**（STA 扫描不关联）；扫描一次 1.4s 会冻屏 → 放后台任务。
- **`nvs_flash_init` 失败不静默擦除**；存档读不到 key=首次运行（默认值），校验失败=显式错误（保留原字节）。
- **按钮回调里不做慢操作**；页面 exit 先停 timer 再删屏（反了定时器访问已删对象）。
- **官方仓库 sdkconfig 会被构建系统重置 WiFi 开关** → 接受"编译进固件但不 init"的现实，运行时不占内存。
