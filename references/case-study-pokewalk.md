# PokeWalk（ESP32-PokemonGo）技术分析报告

> 本文件是《PokeWalk》的完整技术解剖（约 1050 行），作为 ai-passport-game skill 的深度案例参考。主题化提炼见同目录各参考文件；本文件供需要深挖细节（具体代码位置/常量/决策过程）时阅读。

> 分析对象：`D:\aipassport-game\ESP32-PokemonGo-main`（GitHub: Luobata/ESP32-PokemonGo，AI Passport 社区项目 234 / pokewalk）
> 目标：提取一切可复用的嵌入式游戏开发技术与技巧，写给没见过本项目的人照着做。
> 所有结论均来自实际阅读代码与文档，附具体文件路径与关键常量。

---

## 目录

1. [项目全貌](#1-项目全貌)
2. [硬件事实与工程宪法](#2-硬件事实与工程宪法)
3. [自研框架：页面栈与事件循环](#3-自研框架页面栈与事件循环)
4. [渲染引擎：横带软渲染](#4-渲染引擎横带软渲染)
5. [三键交互设计](#5-三键交互设计)
6. [游戏系统设计（S1~S33）](#6-游戏系统设计)
7. [400KB SRAM 上的状态压缩技巧](#7-400kb-sram-上的状态压缩技巧)
8. [数据资产管线（Python 侧）](#8-数据资产管线)
9. [Web 同源预览（最重要的开发效率技巧）](#9-web-同源预览)
10. [存档系统与 USB 备份](#10-存档系统与-usb-备份)
11. [音频系统](#11-音频系统)
12. [Wi-Fi 感知玩法](#12-wi-fi-感知玩法)
13. [构建 / 烧录 / 分区 / 增量更新](#13-构建烧录分区增量更新)
14. [社区发布流程](#14-社区发布流程)
15. [验证文化与门禁体系](#15-验证文化与门禁体系)
16. [工程方法论（最值得进 skill 的部分）](#16-工程方法论)
17. [可复用技术清单速查表](#17-可复用技术清单速查表)

---

## 1. 项目全貌

### 1.1 一句话定位

在一台 **FoloToy AI Passport**（ESP32-C3 掌机，240×320 彩屏 + 三按键）上做宝可梦风格的真实世界探索游戏：设备揣兜里后台 Wi-Fi 扫描、静默累积遭遇；玩家偶尔掏出，30 秒内处理一批。形态接近 Tamagotchi + 宝可梦步行者，不是掌机。

**核心玩法链**：冷启动 P0 开场（7 框打字机 + 大木博士立绘，只播一次）→ P9 选初始伙伴 → P1 待机（三条轴+等级+亲密度）→ P2 遭遇列表 → 圆形转场 → P3 自动战斗（front sprite + 经验结算）→ P4 捕获（进队伍/仓库）→ P5 照料/进化 → P6 图鉴 → P15 路线探索 / P16 森林秘境（roguelike）→ P13 八道馆+四天王+赤红 → 拔电不丢。

### 1.2 目录结构

```
ESP32-PokemonGo-main/
├── firmware/                 # ESP-IDF 5.5.3 工程
│   ├── partitions.csv        # 分区表（见 §13）
│   ├── components/bsp/       # 上游 FoloToy BSP（屏/键/音频/电量/显示）
│   ├── bootloader_components/recovery_boot_hook/
│   └── main/                 # 161 个 .c/.h，约 15.7 万行（含生成数据头）
│       ├── main.c            # 入口：init + demo 菜单 + 按键分发
│       ├── nav.c/h           # 页面表 + 页面栈（自研，非 LVGL 页面）
│       ├── screen.c/h        # 横带帧缓冲 + 推屏 + 截图
│       ├── render.c/h        # 中文字库 + 2bpp sprite 绘制
│       ├── render_scene*.h   # 双端共享 C 渲染配方（固件+Web 同源）
│       ├── world.c/h         # 后台游戏状态唯一所有者（FreeRTOS 任务）
│       ├── play_*.c          # 16 个页面（P0~P16）
│       ├── battle*.c/combat*.c/capture.c  # 战斗子系统（纯逻辑层）
│       ├── sensing.c/h       # Wi-Fi 指纹感知（纯逻辑层，Q10 定点）
│       ├── encounter.c/h     # 确定性遭遇队列 + 图鉴位图
│       ├── party.c/h         # 队伍 6 + 仓库 151（mon_t 12 字节）
│       ├── nurture.c/h       # 三轴养成（Q10）
│       ├── save.c/h          # NVS 存档（V5→V17 前缀保持式版本链）
│       ├── sfx.c/sound_mixer.c/music.c/cry.c  # 实时合成音频
│       ├── assets.c/h        # 嵌入资产解析（gen1/front/back/palettes/font/moves/ui）
│       ├── usb_backup.c + restore_journal.c   # USB 存档备份协议
│       └── dbg.c             # 串口按键注入/截图/探针（调试构建）
├── sim/                      # 20 个纯 Python 参考实现（对账基准）
│   ├── sensing.py / gameplay.py / systems.py / orchestrate.py
│   ├── strings.py            # 文案单一来源（KEYS 提示行、VAR_ELEMENTS 宽度表）
│   └── pixelart.py / effects.py / transitions.py
├── tools/
│   ├── pipeline/             # 173 个脚本：convert_* 资产管线 + 135 个 verify_* 门禁
│   ├── inspector/            # Web 验收平台 + 固件同源预览（server.py/native.py/host/）
│   ├── device/               # fw.sh / walk.py / screenshot.py / monitor.py / idf-env.sh
│   ├── save-manager/         # Web Serial 存档备份页 + CLI restore.py
│   ├── release/              # package_firmware.py + 官方校验器
│   └── collector/            # NDJSON 扫描采集工具
├── assets/                   # 19 个文件：7 个 .bin 固件资产 + 12 个 *_sources.json 溯源
├── data/                     # 真机采集数据 + data/pokemon_moves(gold_silver.json 等)
├── docs/                     # 00~13 编号文档 + systems/S1~S33 + pages/P1~P12 + release/
└── reports/                  # 按日期命名的验收报告 + evidence/ 证据目录（截图/JSON 回执）
```

### 1.3 文档体系（docs/ 的组织方式本身就可复用）

- `docs/00-handoff.md` — **重启会话入口**：TODO 状态、踩坑记录、真机操作速查。核心理念：「数字类事实一律给命令不给值」（写死必过期）。
- `docs/STATUS.md` — 按系统的三态清点（PC 侧/固件/真机），19 个系统 × 8 个页面。
- `docs/01-constitution.md` — **设计宪法**（L0 硬约束层，策划落笔前定稿）。
- `docs/02-sensing.md` ~ `09-device.md` — 分层设计：感知/刷怪/玩法/美术音频/工程/路线图/系统/设备。
- `docs/10-framework.md` — **从本项目抽象出的可迁移框架**（纯模块/门禁/观测五问/编译期不变量/并发纪律），与 06 的分工：「06 是本项目的工程实践，10 是可迁移的方法」。
- `docs/11-render-engine.md` — 双端渲染引擎设计六问与逐像素一致性证据。
- `docs/12-firmware-preview.md` — 固件同源预览的使用与验证边界。
- `docs/systems/S1~S33` — 每个系统一份设计文档（遭遇/捕获/战斗/养成/图鉴/存档/进化/闪光/道具/成绩/开场/取名/音频/队伍/转场/道馆/成就/自动存档/编排/探索/自动学招/动画对齐/金银动画轨道/图鉴追踪/仓库交换/僵局保护/战斗流程与补给/金银伤害/练级渠道/探索补给）。
- `docs/pages/P1~P12` — 由 `tools/pipeline/gen_pages.py` 从 `sim/strings.py` 生成（**文档是生成物**，改文案要改单一来源再生成）。

### 1.4 关键数量级

| 项 | 值 | 出处 |
|---|---|---|
| 固件 main 源文件 | 161 个 .c/.h | `firmware/main/` |
| 工具脚本 | 173 个（其中 verify_* 135 个） | `tools/pipeline/` |
| 嵌入资产 | 7 个 .bin 共 158.8 KB（app 分区的 ~5%） | `assets/` + CMakeLists |
| 应用大小 | 0x175cc0 ≈ 1.53 MB（3.25MB 分区内） | `docs/12-firmware-preview.md` |
| 存档 | SAVE_VERSION 17，整块 blob 写 NVS | `firmware/main/save.h:47` |
| 中文字库 | ~727 字形，42.4 KB，16×16 1bpp | `assets/font16.bin` |
| 纯 Python 参考实现 | sim/ 20 个模块 | STATUS.md |

---

## 2. 硬件事实与工程宪法

### 2.1 硬件清单（`docs/01-constitution.md` + `firmware/components/bsp/include/bsp_pins.h`）

| 项 | 值 | 对设计的直接影响 |
|---|---|---|
| SoC | ESP32-C3FH8X：单核 RISC-V 160MHz，8MB 片内 flash，无 PSRAM | 约 400KB SRAM 是全部内存 |
| 可用堆 | **实测 231KB**（物理 400KB 扣静态；砍 BLE 后） | 整帧 RGB565 150KB 装不下 → 必须横带渲染 |
| 堆布局实测 | RAM 101KiB + Retention 113KiB + 10KiB + RTCRAM 7KiB | constitution.md:32-37 |
| 屏幕 | ST7789P3，240×320，4-line SPI，`SPI2_HOST`，PCLK **40MHz**，出厂需反色（无条件发 `0x21 INVON`），复位脚未接 MCU 走 SWRESET | 推屏必须大端 RGB565 |
| 背光 | GPIO21，LEDC PWM 10-bit 5kHz | 可编程调暗（10%~100% 持久化设置） |
| 按键 | **三键共用一个 ADC 引脚**（GPIO0/ADC1_CH0）分压：上 0mV / 下 ~300mV / 确定 ~595mV / 松开 3300mV；不能改内部上拉（45kΩ 精度差三档重叠） | BSP 产出 PRESS/CLICK/DOUBLE/LONG 四种事件 = 12 种输入 |
| 音频 | **ES8311 codec + I2S 全双工**（MCLK=6 BCLK=5 WS=3 DOUT=2 DIN=4），能录音 | 原推导「无 DAC 只能 PWM」是错的 |
| 电量计 | CW2017（I2C 0x63），`bsp_battery_soc()` 直读百分比 | 续航实测不用外接电流表 |
| 电池 | 520mAh（CW2017 profile）；Wi-Fi 扫描峰值 200~300mA 脉冲 | **屏幕不能常亮，游戏不能实时** |
| 分区契约 | app ≤3MB @0x10000；`cardid` @0x356000；`recovery` @0x700000（上键长按 5 秒进入） | 小程序 BLE 安装的强制契约，不能改 |
| 工具链 | ESP-IDF 5.5.3；UI 层 LVGL（非线程安全，跨任务要 `bsp_lvgl_lock()`） | 上游 AGENTS.md 硬规矩 |

### 2.2 宪法的核心论证（可整体照抄到任何掌机项目）

**「电池预算直接决定玩法形态」**：520mAh + 扫描脉冲电流 → 推导出「设备揣兜里、后台低频扫描、静默累积；玩家偶尔掏出 30 秒处理一批」。并列出四条白捡的好处（不需连续动画、不需实时战斗、一天五次×30 秒更适合品类、三键操作压力骤降）。

**「约束不止是限制，还是风格与机制来源」**（constitution.md §1.4）：

| 约束 | 实际带来了什么 |
|---|---|
| 屏幕不能常亮 | Tamagotchi 式后台累积+短会话形态 |
| 显示带宽受限 | 「大色块+低帧率」恰是 DMG 时代原生表达 |
| 无 IMU | 「移动量」指标摇不出来，比步数抗作弊 |
| 无 GPS | Wi-Fi 指纹在室内往往比 GPS 更准，天生离线隐私好 |

**硬件事实的权威来源优先级**（照抄上游 AGENTS.md）：产品规格与实测 → `bsp_pins.h` → BSP 头文件与实现 → 硬件指南 → README/demo。`bsp_pins.h` 是单一事实来源。

**约束写死在策划之前**：constitution 明说「这类项目最常见的失败方式是：策划做出一个很好的游戏，然后发现电池撑不住、三个键点不动、flash 塞不下。因此硬件约束必须在策划落笔之前定稿」。

---

## 3. 自研框架：页面栈与事件循环

### 3.1 总体分层（`firmware/main/`）

```
纯逻辑层（宿主可编译，0 平台 include）
  encounter.h battle.h capture.h exp.h party.h nurture.h sensing.h
  opening.h transition.h audio.h evolution.h ...
  → 只 include stdint/stdbool；常量放头文件；纯函数无静态状态
  → tools/pipeline/verify_*.py 用本机 cc 编译这些 .c 与 Python sim 逐值对账

平台层（只能设备上验）
  world.c（FreeRTOS 任务+锁+NVS） nav.c screen.c render.c play_*.c（LVGL timer）
  → 只做「搬运与调度」，不含可对账的判断逻辑
```

判据很硬（`docs/10-framework.md` §2.2）：**纯逻辑层头文件的平台 include 数必须为 0**，这是可机械检查的。九个纯模块头文件实测平台 include 全为 0、总 include 1~3 个。

纯模块头文件模板（10-framework.md §2.3，可直接照抄）：

```c
// mymodule.h —— <一句话说清算什么>
// ## 接口约定（不要删这段）
// 本头文件不得 include 任何平台头（esp_*、freertos/*、driver/*、nvs_*、UI 框架）。
// 宿主门禁 include 本模块的 .c 直接编译，一旦引入平台依赖，门禁立刻编不过——那是有意的保护。
#pragma once
#include <stdint.h>
#define MY_LEVEL_MAX 100   // 常量放头文件，门禁要对账常量本身
uint32_t my_curve(uint8_t n);          // 纯函数：同输入必同输出
uint8_t  my_inverse(uint32_t v, uint8_t cap);  // 暂时没用的参数也保留（防后人删掉接口）
```

三条踩过的关键细节：
- **常量放头文件**不要藏 `.c` 的 `static const`（「除数抄错」只有对常量对账才抓得住）。
- **纯函数不要静态状态**，有状态部分留平台层。
- **暂时没用的参数保留**（`exp_to_level(exp, cap)` 的 cap 契约里标注「不可删」）。

### 3.2 页面表与跳转（`firmware/main/nav.h` / `nav.c`）

不用 LVGL 的页面系统，自建一张**显式页面表**：

```c
// nav.c:28 —— 顺序必须与 page_id_t 一致（用下标索引，错位不会报错）
static const page_t PAGES[PAGE_COUNT] = {
    [PAGE_IDLE]  = {"P1 待机", play_idle_enter,  play_idle_exit,  play_idle_key},
    [PAGE_BATTLE]= {"P3 战斗", play_battle_enter,play_battle_exit,play_battle_key},
    ...  // 16 个页面
};
```

每个页面只有三个函数：`enter / exit / key(btn, ev)`。生命周期契约（nav.h:16）：`nav_go(P)` 会 `当前页 exit() → 切 s_cur → 新页 enter()`；**exit 必须先停掉自己的 lv_timer 再删屏**（反了定时器会访问已删除对象）。

为什么不用上游 DEMOS 菜单表（nav.h:3-11 写得非常清楚）：菜单式表是「进一项、长按退回」；玩法是**链式**的 P1→C→P2→A→P3→A→P4→抓到→P6，每步带参数。塞菜单表要么全局变量传参要么跳转逻辑摊到各页 key 回调。**用显式页面表 + 一个 nav_go()，跳转关系集中在一处**。

两种导航模式：
- `nav_go(p)` — 链式跳转（清空返回栈）。
- `nav_open(p)` / `nav_back(fallback)` — 菜单下钻，`s_return[RETURN_DEPTH=6]` 有界返回栈，防意外递归链接超深；`nav_is_returning()` 让 enter() 保留上次光标。

### 3.3 页面间传参：nav_ctx 单槽（nav.h:99-118）

```c
typedef struct {
    encounter_t enc;      // 正在处理的这一只（值拷贝）
    uint16_t uid;         // **用 uid 而不是队列下标**认这一条
    bool valid, exploring, battled, battle_won;
    uint8_t done_note;    // 完成页摘要（捕获/进化），来源页短暂停留后自动导航
} nav_ctx_t;
```

理由：链路上同时只有一条在处理，不需要栈。**uid 不是下标**：玩家在 P3/P4 期间后台还在往队列塞、满了淘汰首条会使下标左移，实测症状「打的是 #64，抓到的是 #23」。

### 3.4 事件循环：LVGL timer + 后台任务双世界

- **前台**：LVGL 任务跑 `lv_timer_handler`，每页用 `lv_timer_create(tick, 周期ms, NULL)` 驱动动画（如 P5 的 `motion_tick` 80ms）。按键回调在 button 组件任务里，进 LVGL 前 `bsp_lvgl_lock(500)`。
- **后台**：`world.c` 一个独立 FreeRTOS 任务（栈 4096）做 Wi-Fi 扫描（`SCAN_INTERVAL_MS 30000`）+ 感知 + 养成结算 + 节流存档。**在页面之前起**，因为页面进来就要 `world_snapshot()`，且 world 是 Wi-Fi 的唯一所有者（两个所有者会争射频返回 `ESP_ERR_INVALID_STATE`）。
- 为什么不用 lv_timer 做扫描：一次扫描阻塞 1.4 秒屏幕冻住；且 lv_timer 属于页面，页面销毁定时器就没了。

### 3.5 线程契约写进头文件（world.h:16-77）

world.h 专门有一节 `## 线程契约`：
- world 任务里绝不碰 LVGL，只更新自己的状态。
- 页面读状态走 `world_snapshot()`——互斥锁内拷一份值快照，避免「饱食是新的、心情是旧的」撕裂读。
- **LVGL 锁只保护界面对象，不能替代跨任务状态同步**（这条最重要——原先有个一致性缺陷实际靠 LVGL 递归锁间接保证，头文件一字未提，音频任务一起新就可达）。
- 不可从 ISR 调用（互斥量可能阻塞）。

### 3.6 并发纪律（world.c 实现 + 10-framework.md §6）

- **临界区内只做内存拷贝（微秒级），NVS 写入在锁外**。历史缺陷：锁超时 50ms 的降级分支 vs 同锁下持锁写 flash 最坏几十毫秒——超时值与最坏持锁时长同量级。
- 双锁单向顺序：`s_save_lock → s_lock` 全局一致永不反向（world.c:190→216）。
- 「拿不到锁就读旧值」必须有独立影子副本，否则不是降级是数据竞争。world 曾有「超时偷读」分支，修复时**删掉了**——接口是 `void`，偷读给不出一致性保证。

---

## 4. 渲染引擎：横带软渲染

### 4.1 为什么横带（`firmware/main/screen.h:12-18`）

```c
#define SCREEN_W 240
#define SCREEN_H 320
#define SCREEN_BAND_H 80                    // 240×80×2 = 37.5KB
#define SCREEN_BANDS (SCREEN_H / SCREEN_BAND_H)  // 4 条，80 整除 320 无余数
static uint16_t s_band[SCREEN_W * SCREEN_BAND_H]; // 唯一的帧缓冲（screen.c:43）
```

整屏 150KB = 可用堆 231KB 的 65%，且 static 数据要在**链接期**挤进 dram0_0_seg（比运行期更紧——曾开 150KB static 截图缓冲直接 DRAM 溢出 32624 字节链接失败）。横带 37.5KB 静态缓冲是安全值。

**代价与纪律**：一个元素跨横带边界时，两条带重画频率不同（如待机页带 3 每拍重画、带 2 每 8 拍重画），上下两半不同步 = 撕裂的字。因此有编译期断言（§4.4）。

### 4.2 绘制接口：极简三个函数

```c
uint16_t *screen_band(void);                 // 直接写缓冲（画 sprite 快路径）
void screen_band_clear(uint16_t rgb565);
void screen_px(int x, int y, uint16_t c);    // 坐标是横带内的，越界静默忽略
void screen_push_band(int band_y);           // 推屏（内部处理字节序）
```

页面绘制模式（`play_care.c:91-178` 是标准样板）：

```c
static void draw_band(int band_y) {
    screen_band_clear(GAME_UI_BG);
    #define Y(v) ((v) - band_y)          // 整屏坐标 → 带内坐标的宏
    ... render_text(12, Y(40), buf, GAME_UI_INK); ...
    #undef Y
    screen_push_band(band_y);
}
static void draw_all(void) { for (int y=0; y<SCR_H; y+=BAND_H) draw_band(y); }
```

每个元素坐标在文件头 `#define`，紧跟 `SCREEN_ASSERT_WITHIN_BAND(name, y, h)` 断言（play_care.c:39-63 共 12 条）。

### 4.3 推屏：字节序 + DMA 同步的两个坑（screen.c）

**坑一：字节序**。ST7789 走 SPI 要大端 RGB565，RISC-V 内存是小端。绕过 LVGL 自己推屏必须手动交换。做法是**原地交换**（screen.c:150-153）——所以截图必须取交换**之后**的字节，否则观测手段看不见字节序类故障（第一版在交换前吐，屏幕紫了截图还是绿的，据此误判渲染没问题）。

**坑二：DMA 异步竞争**。只有一块 s_band，`esp_lcd_panel_draw_bitmap` 是异步的（排进 SPI 队列就返回）。四条带连着画：带 0 排队→立刻回来→带 1 重填 s_band→带 0 的 DMA 读到带 1 的像素。实测用户报「紫/绿混杂、底部闪烁」。

三个解法的取舍（screen.c:103-127 注释）：
- 注册 `on_color_trans_done` 回调？**不行**——那个回调槽已被 esp_lvgl_port 占了，一注册就把 LVGL 的顶掉，直接看门狗复位循环。**一个 panel_io 只有一个回调槽，而这块面板与 LVGL 共用**。
- 双缓冲？再要 37.5KB DRAM。
- **最终方案（零内存开销）：用一次 `tx_param` 逼出同步点**。读 esp_lcd_panel_io_spi.c 源码发现任何 polling 传输前驱动会先把队列里在途传输收干净，所以发一条最便宜的命令等于「等前面画完」：

```c
static void wait_dma_done(void) {
    esp_lcd_panel_io_handle_t io = bsp_display_io();
    if (io) esp_lcd_panel_io_tx_param(io, 0x00, NULL, 0);   // NOP，几微秒
}
```

### 4.4 编译期布局不变量（screen.h:24-39）

```c
#define SCREEN_ELEMENT_FITS_BAND(y, h) \
    ((h) > 0 && (y) >= 0 && (y) + (h) <= SCREEN_H && \
     (y) / SCREEN_BAND_H == ((y) + (h) - 1) / SCREEN_BAND_H)
#define SCREEN_ASSERT_WITHIN_BAND(name, y, h) \
    _Static_assert(SCREEN_ELEMENT_FITS_BAND((y), (h)), \
                   #name " crosses a screen band boundary")
// 显式允许跨带（sprite 用）：高度 ≤ 2×带高，且必须重画所有涉及的带
#define SCREEN_ASSERT_ALLOW_CROSS_BAND(name, y, h) ...
```

项目落了 14+ 处。起因是用户实测发现 P1「那片区域有时先渲一半再出来」——不变量只活在 `play_idle.c` 文件头注释里，没有任何机制检查。**可机械检查的不变量要写成断言；「元素不跨带」是纯算术，完全可以在宿主上算**。注意：不是所有跨带都要修——sprite 有意跨带 0/1 且那两带同频重画，断言只加在「要求单带刷新」的元素上。

运行期兜底：`screen_px` 越界静默丢弃——10-framework.md 建议**任何静默丢弃路径都应可计数**（调试构建加计数器）。

### 4.5 与 LVGL 共存：screen_own_display()（screen.c:62-80）

游戏不建 LVGL 对象直接画横带，但 LVGL 任务还在跑。坑：`lv_screen_load()` 空屏会**排队一次整屏刷新**，它在游戏首帧后执行就会用空背景盖掉游戏横带（待机页只重画部分带，y=160-239 留下永久绿条）。解法：在 LVGL 锁内、首次游戏绘制前：

```c
lv_display_enable_invalidation(display, false);   // 关闭失效标记
lv_timer_pause(lv_display_get_refr_timer(display)); // 暂停该 display 的刷新定时器
```

LVGL 主任务、游戏动画定时器、按键、休眠计时器照常运行。配套门禁 `verify_display_ownership.py` 编译真实 LVGL + 生产 screen.c，模拟面板输出边界：旧行为 16 次延迟背景刷新，修复后 0 次（负向对照：移除保护必须再次检出）。**串口截图会主动重画当前页，捕获的是游戏像素，看不见 LVGL 后来写入的覆盖——所以这类问题截图证明不了**。

### 4.6 截图通道（screen.c:174-270）

- `screen_dump()`：`@@SHOT 240 320 rgb565be 80` 头 + 每条带 base64（每行 76 字符防串口缓冲丢行）打到串口，`tools/device/screenshot.py` 收成 PNG。**回调式不缓存整屏**——150KB static 曾链接失败，代价是 dump 时画面闪一下。
- `screen_dump_fap()`：官方发布用二进制截图 `FAP_SCREENSHOT_V1 240 320 RGB565LE 153600`，持 LVGL 锁回放当前帧、屏蔽后台日志、控制行结尾。
- **两条硬边界**（00-handoff.md 写明）：
  1. 截图抓不到任何动画中间态——`screen_dump` 调的是页面 `redraw_for_dump`（按当前状态变量重画），不是抓那一帧；且 150KB base64 200KB 传输 ≥2.8 帧时间。帧级时序用宿主对账（verify_opening.py 逐帧比状态机）或串口帧号轨迹。
  2. 截图能证明「画出来了」，不能证明「用的是哪个数据源」（Gen1 back 在低分辨率下看着也像正面）——换 sprite 要**两张图对比 + bbox 度量**（项目用过 40×32 → 74×66 判据）。

### 4.7 字体渲染（render.c）

**字库格式**（`convert_font.py` 定义，render.c:39-41 注释）：
```
头部 14B: magic "FNT1" | ver u16 | size u16 | per u16 | count u32
码点索引: count × u16，升序（可二分查找）
字形数据: count × per（16×16 1bpp = 32B/字，每行 2 字节高位在左）
```
解析时强校验：magic、长度自洽 `14 + cnt*2 + cnt*per == len`；**不写死 count/per，从头读**。

**半宽 ASCII 的规格冲突解法**（render.c:3-26 大段注释）：`sim/strings.py` 的 text_px() 按「汉字 16px、ASCII 8px」算，但字库是定长 16×16 无 advance 字段。三条路中选「**渲染器按字符类型决定步进**」：`cp < 0x80` 时 advance = size/2 = 8px，并把字形**左移 size/4 = 4px** 再画（字形墨迹居中在 16 格里，A 在 x=2~11，按 8px 步进只切掉右侧空白不动墨迹）。零字库改动、零格式变更，一个 `cp < 0x80` 判断比给 650 字形各加 1 字节 advance 划算。

**自检即文档**（render.c:261-311）：启动时 `render_selftest()` 验码点索引升序（二分依赖它）+ 7 个探针字符（每个都是历史上真缺过的：「鉴」曾因 HK 字体全空、「A」「[」曾整个缺失）+ 两个排版锚点（`"[A]照料"`=56px、P1 提示行=184px 与 sim 逐值一致）。**探针挑各系统的关键字，缺任何一个都说明字库与代码不同步**。

### 4.8 Sprite 渲染（render.c:210-257）

- 2bpp 格式：每字节 4 像素高位在左，色号 0~3 查 4 色 RGB565 调色板，**3 号透明**。
- 最近邻整数倍放大（`render_sprite_2bpp_wh(x,y,data,w,h,scale,pal)`）——「双线性会把硬边缘糊掉，而硬边缘正是像素风的一部分」。
- 缩略图降采样 `render_sprite_2bpp_thumbnail`：`sy = ((2*dy+1)*size)/(2*dest_size)` 中心采样，图鉴 40/48/56px → 32px 缩略用。
- `render_sprite_ink_bounds`：扫描可见轮廓 bbox——战斗页进场扫一次轮廓做对齐（可见轮廓顶边 y30、右缘 x232 统一），避免源素材透明留白造成错位。
- 调色板读取集中在 `assets_palette/variant`（assets.h:53-60）——「提到这里而不是各页自己读：三份拷贝里只要有一份把偏移算错，那一页颜色错而另外两页正常，极难注意到」。shiny 变体偏移用 size_t 计算防 8 位下标回绕。

### 4.9 双端共享渲染配方（render_scene.h / render_scene_screen.h + docs/11）

解决「D28 呈现层单一来源」架构债的方案（值得整段学习）：

- **共享 C 场景配方**：输入页面只读状态快照，输出有序绘制指令（目前原语 `rect(x,y,w,h,rgb565)` 与文本回调）。布局、整数比例、颜色只在 C 里算一次。
- 固件侧 `render_scene_screen.h`：绝对坐标与目标 band 求交、y 减 band_y、调既有 screen_px——复用横带缓冲，不申请帧缓冲。
- Web 侧：构建期原生执行同一 C（scene_cli）导出矩形 payload，JS 后端只做裁剪+写 Uint16Array，**不再有第二份求解**（web 的 `Math.round((w-2)*pct/100)` 与固件 `w*cur/max` 整数除法曾产生像素级差异——「仅共用 x/y/w/h 不能保证像素相同」）。
- 验证方式：125 组输入 × 9600000 RGB565 像素逐值一致（native 横带后端 vs 旧实现 vs JS 消费端），负向对照故意把 payload 每个矩形 x+1 必须检出首差。sprite 迁移时用 151 张真实 BACK 图 × 5 个抖动相位 = 755 次整帧比对。
- **预算桥**：`tools/pipeline/gen_layout_budget.py` 从 `sim/strings.py` 的 VAR_ELEMENTS 表生成 `render_layout_budget.h`（如 `LAYOUT_P3_PET_NAME_BUDGET = 120`），`--check` 模式检测表与头不同步（陈旧 exit 1）。消除「坐标/预算在 Python 表和 C 里各写一遍」的漂移。
- **测宽解耦**（D92）：`render.h` 内联定义唯一一套 `render_utf8_next / render_char_advance_sized / render_text_width_sized`，宿主只 include 头文件即可求值，不需要 ESP 日志或链接 render.c。655360 组步进一致 + ASan/UBSan 通过。

### 4.10 帧率策略

没有 60fps。tick 周期按内容定：照料动画 80ms、战斗回合「抖 4 帧 + hold 6 拍 = 600ms」、技能阶段最长 18×60ms、40 回合上限约 43.2 秒（验收 walk.py 要 `--settle 46`）。脏区按**横带**为单位重画（如 P3 技能阶段固定重画前三条带，消息区只在换回合重画）。这与「大色块+低帧率是 DMG 原生表达」的宪法一致。

### 4.11 颜色系统（screen.h:52-60）

九色 RGB565 实值常量，来源标注清楚（GSC UI colors: bg_tiles.pal, hp_bar.pal, exp_bar.pal）：`C_BG=0xFFF0 / C_INK=0x0000 / C_MID=0x7240 / C_LIGHT=0xAD55 / C_HP_TRACK=0xF6AF / C_FOCUS=0x247F / C_HP_GREEN=0x05E0 / C_HP_YELLOW=0xFD60 / C_HP_RED=0xF800`，配 `RGB(r,g,b)` / `RGB_HEX(h)` 宏。HP 三段**按填充像素判不按百分比**（`fw >= w*50/100` / `fw >= w*21/100`，照 asm `GetHPPal` 的像素判据形状）——「条宽改了不用重推阈值」（同一规则三处三个值是本项目踩过的真实坑：asm 像素判据 / web `>21%` / 契约 `>20%`）。

---

## 5. 三键交互设计

### 5.1 全局手势分配（nav.h:60-75 + main.c:102-135）

BSP 把三键 ADC 分压解析成 `bsp_btn_t {UP, DOWN, OK}` × `bsp_btn_ev_t {PRESS, CLICK, DOUBLE, LONG}` = 12 种输入。项目把语义收敛成一组 inline helper，**所有页面共用**：

```c
static inline int nav_direction(btn, ev)  // CLICK: UP=-1, DOWN=+1（光标移动）
static inline bool nav_confirm(btn, ev)   // OK + CLICK（确认）
static inline bool nav_return(btn, ev)    // DOWN + LONG（返回）
static inline unsigned nav_list_selection(btn, ev, count, current)
    { return count ? (current + count + nav_direction(...)) % count : 0; }  // 环形
static inline bool nav_list_activate(btn, ev, count)
    { return count && nav_confirm(...); }
```

即：**A(上)短按=上一项，B(下)短按=下一项，C(确定)短按=确认，B(下)长按=返回**。页面 key 处理的标准形状（play_care.c:236-244）：

```c
void play_care_key(bsp_btn_t btn, bsp_btn_ev_t ev) {
    if (nav_return(btn, ev)) { nav_back(PAGE_IDLE); return; }
    if (nav_direction(btn, ev)) {
        ...刷新快照...; s_sel = (s_sel + action_count() + nav_direction(btn,ev)) % action_count();
        draw_all(); return;
    }
    if (!nav_confirm(btn, ev)) return;
    switch (btn) { case BSP_BTN_OK: ...执行动作...; }
}
```

**长按不额外触发短按**的实现不在项目代码里，而在 BSP 的行为 + nav.h:59 的注释：「DOWN long is exclusive of DOWN click (**the BSP suppresses the click after a completed hold**)」——BSP 在长按完成时抑制随后的 click 事件，nav 的 helper 只认 CLICK，于是天然互斥。另外 `nav_direction` 只在 `ev == BSP_BTN_CLICK` 时返回非零，LONG/PRESS/DOUBLE 一律 0，双击不会移动光标。

### 5.2 全局占用的手势（写过文档防重踩）

- **C 长按（OK+LONG）被 main.c 拦截作熄屏/显示电源**，不转发 `nav_key()`——「任何页面都不能用它」（00-handoff.md 惯例节，2026-09-06 曾给 P2 安排「C 长按丢弃」，实际那个手势到不了页面）。
- 输入过滤层：`screen_idle_filter_key()` 统一处理熄屏唤醒，`main.c:111` 在 demo 壳层过滤，`nav.c:120` 在游戏层过滤——同一逻辑两处入口。
- 重操作配重手势：「永久删除、无确认、无撤销的丢弃」从单击移到 **B 长按**（重操作要重手势）。

### 5.3 「一键一事」与提示行治理

交互 review 修掉的核心缺陷全是「UI 承诺的与实际行为不符」：P2 提示行写 `[A]选中` 但按 A 只移光标（选中是未标注的双击）——**阻断级**，改成 `[A]选中 [B]下条 [C]返回` 一键一事；死按钮（`[A]详情` 只打串口）改成「上页」双向翻页；静默失败（球用完按 A 屏幕零变化）改成屏显「没有球了」。

提示行约束：**单行、每个 action ≤ 2 字**（`sim/strings.py` 的 `KEY_HINT_MAX_ACTION = 2`，`check_key_hints()` 审计宽度 ≤232px）。「想表达『单击 X / 双击 Y』表达不出来——该改的是交互（让一个键只做一件事），不是文案格式」。新页面提示行必须进 `KEYS` 表（单一来源），不硬写在固件里。

### 5.4 完成后自动导航的判据（STATUS.md 用户反馈节）

「捕获成功就停住没有下一步」→ 查全部 14 处 `nav_go`：只有 1 处由「事情完成」触发。**判据：玩家在这一刻有没有选择可做？没有就自动跳，有就留在原地改提示行**。反面表述：如果自动跳会剥夺玩家一个真实的决定，就不要自动跳。按这条判：捕获成功自动回 P2、进化完成自动回 P1、**战斗打完不跳**（「打完去抓」是真实选择）。

实现不新造基础设施——项目已有跑通模式（`play_enc.c` 的 transition_tick：播完→清状态→nav_go），抽成规则 + `nav_ctx` 加 `done_note` 字段。三个容易漏的细节：
1. 先清状态与 timer 指针再 nav_go（否则 exit() 重复删 timer）。
2. 停留期按键只把计数推到最后一拍，不直接 nav_go——**跳转只有一条路径**。
3. nav_go 必须在 LVGL 任务里调（lv_timer 回调天然合规，不能从 world 任务或 ISR 调）。

真机验证含负例：提前按键 124.9ms 跳过 ✓、P3 完成后观察 1.25 秒确认不自动跳 ✓。

### 5.5 熄屏与输入

`screen_idle.c`：无操作自动熄屏计时；`nav_screen_busy()` 区分「有限动画与定时结果保持亮屏」vs「输入选择、待机呼吸、后台刷新不阻止超时」。`screen_idle_input_dropped()` 处理拿不到 LVGL 锁时丢键的记账。熄屏期间扫描降频 30→60 秒（社区版 2026-09-15 的待机省电）。

---

## 6. 游戏系统设计

docs/systems/ 共 40 份文档。以下是按「可复用设计模式」提炼的核心。

### 6.1 确定性遭遇（S1，encounter.h/c）

**核心决策：整个遭遇系统是确定性的，靠 crc32 而不是随机数**（encounter.h:5-12）：

- `enc_spawn_seed(bssid, ts)` — 同一 AP 同一小时永远同一个种子。
- 好处：跨设备一致（两人站一起看到同一批怪，不需要服务器）、可预测（「这个 AP 下午三点出火系」能做攻略）、可重算（不用存刷新表）、**能与 PC 侧逐值对账**（不像战斗只能对分布）。
- `enc_rarity_from_ap(rssi, auth, has_ssid, is_transient)`：AP 属性 → 稀有度 1~5。信号弱/隐藏 SSID/企业级/瞬现都加分——「玩家追着奇怪的路由器跑这件事本身就很对味」。
- `enc_pick_species`：按稀有度从对应强度档挑——**不是均匀采样 151 只**（均匀采样会让 Lv12 主宠遇到超梦，实测打 46 回合赢不了）。
- 闪光 `enc_roll_shiny`：crc32 确定性判定，极稀有（rarity≥5）概率翻倍（1/512 基准）。

**队列**（encounter.h:19-51）：物理 16 槽（`ENC_QUEUE_CAP 16` 保存档布局），玩法可见 5 条（`ENC_QUEUE_LIMIT 5`）；满后无条件淘汰首条；`next_uid` 单调递增（0 保留作无效）；`dropped` 统计被挤掉数。一条 `encounter_t` 16 字节：uid u16 / level u8 / activity u8 / ts u32 / species_id u16 / rarity u8 / biome u8 / hp_ratio u8 / is_shiny / is_transient / exp_granted 三个 bool。

**双路径供给**（「让时间生产内容，让空间调制内容」，02-sensing.md §2.0）：基地遭遇按时间排程（`ENC_BASE_INTERVAL_S`，4 小时一次），猎场遭遇靠瞬现 AP（移动中）——任何「见到新 BSSID」指标都会衰减到零，所以驻留也要有产出。

### 6.2 自动战斗（S3 + battle.c/combat.c）

回合制自动结算。回合 = 抖 4 帧 + hold 6 拍 = 600ms；`BATTLE_MAX_ROUNDS 40` → 最坏 24 秒（后扩到 43.2 秒）。金银伤害公式（S31/S32）带 Q10 系数缩放（factor 614/1024 ≈ 0.6 的消沉惩罚）。相克表 225 格逐值对账（verify_battle 791 组）。僵局保护（S29-stalemate-protection.md）。战斗种子可被 dbg.c 覆盖（`dbg_battle_seed`）——设 31 让第一回合必 miss，用来截「的攻击落空了！」（miss 是随机事件，不注入截不到）。

### 6.3 养成三轴（S4，nurture.h）

Q10 定点四轴：`satiety`（饱食）/ `mood`（心情）/ `stamina`（体能）/ `intimacy`（亲密度），加 `last_us` 上次结算时刻。

常量（nurture.h:37-58，与 sim/gameplay.py 逐个对应）：
```
饱食衰减 4/小时   心情衰减 3/小时   体能恢复 100/小时（后调成约1小时回满）
亲密度 0.5/小时   喂食 +30饱食+5心情   玩耍 +15心情−5体能+1亲密
战败 −20体能−15心情   消沉阈值：饱食或心情 < 25
```
时间源 `esp_timer_get_time()` 开机微秒——**它会漂**（CONFIG_RTC_CLK_SRC_INT_RC 内部 RC ±5% 温漂，实测 +137ppm），对养成无感但影响日切判定（S10 至今未做的原因）。关机期间的衰减后来靠「Wi-Fi 校时 + UTC 锚点 + 离线体能补算」解决（save V17 加 rest_clock，社区版 804）。

### 6.4 经验与追赶（S32）

- 升级总需求 `floor(3 × level³ / 2)`（exp.c 纯函数，651 组逐值对账；`exp_for_level(100) = 1,500,000` 远小于 uint32 上限）。
- 单场奖励 `max(level×8+20, ceil((下一级总需求−当前级总需求)/8))`——20 级后约 8 场同级胜利升一级（旧公式 Lv99 要 55 场）。
- 胜利 100% / 失败 30% / 捕获 60% / 首捕获该物种 +50%。
- **队伍追赶**（防「只有主宠在涨」）：取队伍最高等级 H 与成员等级差 G——参战成员先平分再乘 `100+min(100, floor(200·G/H))%`（最高 200%），未参战各得 `20+min(80, floor(160·G/H))%`（最高 100%）；加成随差距缩小自动退回；最高等级在整次发奖前固定，不受遍历顺序影响。野战/捕获/训练家/探索共用 `exp_award_party`。
- 幂等：`encounter_t.exp_granted` + 锁内按 uid test-and-set。修的真实缺陷：「打完→逃跑→再打」可无限刷经验（on_select 只拷贝不取走 + play_battle_enter 重置 s_done）。

### 6.5 图鉴与闪光（S5/S8）

**四张位图而不是两张**（encounter.h:55-60）：`seen / caught / shiny_seen / shiny_caught`，各 19 字节（151 物种位压缩）——「闪光遇到了但跑了」那份遗憾要能存下来。图鉴页 5×4 每页 20 只共 8 页；已捕获彩色、见过未捕获用同一正面图单色剪影、未见过只显示编号。

### 6.6 道馆与训练家战役（S16/S17）

线性八馆+四天王+赤红；门槛与真实数据强相关（第 6 馆「商业区 4 次」按真实采集只占 0.8% 几乎不可达——**稀有 biome 不能用累计小时衡量**的教训）。训练家战役是事务性的（world_challenge_* 系列，WORLD_CHALLENGE_BUSY/LOCKED/SAVE_FAILED 等结果枚举），战斗会话 `battle_session_t` 按 uid 存运行时（不进存档，重启即失，但「已开战」的移除先落盘防重入）。数值纪律：**不手填，从 pret 反汇编取**（data/trainers/parties.asm；曾凭记忆手填坂木错两处阿渡错一处）；中文名用**任天堂官方译名** zh_nin（徽章是颜色名：灰色徽章而非岩石徽章，从英文回译八个全错）。

### 6.7 探索与地图（S20/S26/S33）

9 章节 36000 步的路线探索（森林/山洞/海滩/电站四张地图，属性匹配：森林草/虫、山洞岩石/地面、海滩水/冰、电站电/毒）。图鉴追踪三段线索循环（零线索约七次探索出目标，换目标清旧线索防挪用）。地图普通训练家 12 位（每图三位三档，徽章 0/2/5 解锁，敌方等级 = 队伍最高 90% + 档位补偿，封顶 `12+7×徽章数`）。每路线研究任务（登记 5 种/捕获 3 种/完成一次追踪）。

### 6.8 森林秘境 roguelike（S26 系列 + dungeon.c/h）

`dungeon_t` 三层版本前缀保持（v1: seed/xp/cards/ids[3]/choices[3]/wins/guards/berries/stone → v2: run_id/earned/receipt/relay_mask → v3: 换成自有队员 members[3]+slots）。15 张卡（`DUNGEON_CARD_COUNT 15`）、8 种节点（EMPTY/BATTLE/REWARD/FORK/CAMP/WON/LOST/SETTLEMENT/ENTRY）。进秘境**预约占用队伍槽**（`dungeon_party_locked()`，world 侧换宠锁 switch_locked）——roguelike 与主存档的边界靠「槽位预约 + 收据 receipt + run_id 幂等」管理。退出确认防误触。验收含 `verify_dungeon_stack.py`（栈深度）与 `verify_dungeon_device.py`。

### 6.9 「完成→导航」与转场（S15）

八种转场几何（圆形/百叶等，134400 格逐格对账 + 32 组选型含 uint8 溢出边界），P2→P3 接圆形。转场 tick 播完清状态再 nav_go（见 §5.4）。

### 6.10 自动学招（S21）

`assets_known_moves(species, level, out, max)`：按等级全量招式表**现算**，含进化前祖先招，去重按 move ID 排序——不存招式列表进存档（「现算比存档省」）。无四招上限不耗 PP（设计选择）。55 招专属化：46 个 ID 定向映射 + 15 属性共享效果兜底。

---

## 7. 400KB SRAM 上的状态压缩技巧

这是「如何在 400KB SRAM 设备上设计完整 RPG」的答案汇总：

1. **mon_t = 12 字节**（party.h:11-16）：`species_id, level, hp, intimacy` 各 u8 + `explore_value` u16 + `nickname_idx, flags` u8（flags bit0=shiny）+ `exp` u32。昵称不存字符串存**索引**（S12 预设名池）。队伍 6 + 仓库 151 格 = `PARTY_BYTES = 2 + 157×12 = 1886` 字节。
2. **图鉴 76 字节**：4 × ⌈151/8⌉ = 4 × 19 字节位图。
3. **遭遇 16 字节/条**，物理 16 槽，玩法 5 条。
4. **Q10 定点代替 float**（C3 无 FPU）：感知权重、养成轴、伤害系数全是 Q10（×1024）；标度选择论证见 sensing.h:14-18（Q8 误差 0.002、Q10 误差 0.0007 且 u16 装得下、Q16 白白翻倍宽度）。
5. **大数组必须 static 不入栈**：world.c 文件头记着 3584 主栈被 `sens_core_t`（2.6KB）+平滑窗口（1.9KB）撑爆 Guru Meditation 的历史；现在扫描结果 `s_recs[64]`（64×76B）、感知核心全 static。存档缓冲 `s_save_buf`（约 3.7KB）也 static——「save_t 含 1886B 队伍区，不能放进 4KiB 的 world task 栈」。
6. **布隆过滤器代替集合**：`SENS_BLOOM_BITS 8192`（1KB）记「见过的 AP」，实测三份数据 149 个 BSSID 只占 3.6% 的位，与真集合零差异。
7. **LRU 定长槽代替数据库**：地点记忆 8 槽（`SENS_PLACE_SLOTS 8`，512 字节预算），每槽 48 字节加权指纹 + 元数据。
8. **音频零 PCM**：音乐/叫声/音效全部运行时合成（音符数据 164B 起步），不存采样。
9. **资产进 flash 不进 RAM**：CMake `EMBED_FILES` 直接嵌 app 分区 158.8KB，解析函数逐字节读（`rd16/rd32` 不强转指针——RISC-V 未对齐访问是异常不是静默慢速）。
10. **横带渲染 37.5KB** 代替整帧 150KB；截图回调式零常驻。
11. **砍组件换堆**：砍 BLE+多余 demo 从 187KB 换回 231KB（44KB 在 C3 上是一大笔）。
12. **存档整块 blob**：约 3.7KB 一次 `nvs_set_blob`，避免 8 次写入 8 个失败点（save.c:40-46 论证）。

---

## 8. 数据资产管线

### 8.1 总体流向

```
上游真源（不手填、不手画）
  pret/pokered + pokecrystal 反汇编（gfx/trainers/prof.oak.png 等，固定 commit）
  PokeAPI（gray 变体，colortype=0/depth=2 真四阶灰）
        │ fetch_gen1.py / fetch_oak.py / fetch_balls.py / data/pokemon_moves/gold_silver.json
        ▼
assets/*_sources.json（19 个溯源文件：pokemon_art_sources.json 255KB、
        pokemon_animation_sources.json 390KB 等——每个像素/音符都能追到上游坐标）
        │ convert_*.py（convert_sprites/palettes/gen1/font/moves/ui/cries/music/
        │   battle_fx/pokemon_animation/route_trainers/trainers/party_balls/...）
        ▼
assets/*.bin（7 个，158.8KB，EMBED_FILES 进固件）
        │ inventory_assets.py（资产断言，退出码当 CI 门禁）
        ▼
固件 assets.c 解析（magic + 长度自洽校验，REC_SIZE 从文件头读不写死）
```

### 8.2 二进制格式（小端，头部自描述）

| 文件 | magic | 头 | 记录 |
|---|---|---|---|
| gen1.bin (7.3KB) | `GEN1` | 16B: magic+ver u16+rec_size u16+count u32+pool_size u32 | 32B 定长/只 + 名字池（zo/zl 偏移长度在记录 20/22） |
| gen1_front.bin (90KB) | `FRNT` | 分段表 | 三档 40/48/56px，记录步长 `2+per`，显式物种 id |
| gen1_back.bin (87KB) | `BACK` | 宽/高/per 从头读 | 32×32 与 48×48（Crystal）共存，步长不写死 256 |
| font16.bin (42KB) | `FNT1` | 14B: magic+ver+size+per+count | 升序 u16 码点索引 + 32B/字 1bpp |
| moves.bin (4KB) | — | 16B | 招式表 |
| palettes.bin (2.5KB) | — | 12B 头 + 8B/组 | 135+ 套 4 色 RGB565，shiny 配对偏移 |
| ui.bin (4.4KB) | — | — | 精灵球/光标/心/星等 UI 点阵，**按名字查**（个位数素材线性查找 < 下标改了忘同步的代价） |

解析纪律（assets.c 头注释）：**每个 REC_SIZE 从文件头读，不写死**——convert_gen1.py 注释曾把 32 写成 28，照抄会让第 2 条记录起错位 4 字节，而错位后读出的仍是合法数值（种族值都在 0~255），不崩只是全错。每个 parse 校验 `头 + count×rsz + pool == 文件长` 自洽，不対就整表置空。`assets_selftest()` 把关键数字打到串口与 `inventory_assets.py` 对账。

### 8.3 中文字库子集化（convert_font.py，415 行）

- **只打包实际用到的字**：字符集来自八个上屏来源（物种中文名 209 字起步 → UI 文案/昵称/开场台词/道馆/队伍菜单/判定提示/存档槽 → 现 ~727 字形 17.6~42KB）。
- **16×16 而非 12×12 的论证**：实测「妙」12×12 着墨 20% 糊成一团、16×16 着墨 23% 左「女」右「少」清晰可辨；209 字体积差 2.8KB，「清晰度值这 2.8KB」。
- **字体集合必须指定子字体 index**：PingFang.ttc 12 个子字体，Pillow 默认 index 0 是**香港繁体**——用它渲简体的实测后果：「鉴」全空白（HK 面没这字形，P6 标题就是它）+ 142 个字形港台写法（27% 的字库是错的，而字数/文件大小/缺字差集统计全正常）。修法：`FONT_CANDIDATES` 传 `(路径, index=2, "PingFang SC Regular")` + 子字体自检字 `SANITY_CHARS = "鉴龙"`（「鉴」验证简体存在、「龙」验证不是繁体「龍」）。
- **扫源码字面量 + 构建自动同步**：`_firmware_chars()` 扫 `firmware/main/*.c` 字符串字面量收汉字（跳过注释）。**宁可多收不少收**：多收一字 32 字节，少收一字是真机空白+重烧。`fw.sh build` 每次自动跑 convert_font（代价 2 秒 PIL）——「加一句中文日志就可能引入新字」这类失败被构建吸收。
- **名字从入库产物读不从中间文件**：`_names_from_bin(gen1.bin)`——/tmp 中间 json 被清过一次，字库从 420 字掉 297，中文名全丢。
- **字形不存在的字符（▸ ♥ ✦）不进字库**，改用 `pixelart.py` 算法生成点阵进 ui.bin——「PingFang 全系列无这三个字形，收进字库只会得到隐形符号」。

### 8.4 转换管线的三条铁律（10-framework.md 第七章）

1. **同一条管线单一来源**：所有素材从同一上游走同一转换。「手画规则几何体都失败过，人物风险高一个量级」——手绘大木博士被用户一眼否决（「和皮卡丘不是一个风格」），改用 pret 反汇编真品 56×56 depth=2 与 front sprite 同格式同管线一次成功。放大只能**整数倍**（非整数倍插值糊掉点阵）。
2. **同一编码多重语义显式处理**：2bpp 色号 3 在精灵外=透明、精灵内=白（高光）。不做区分的后果：Oak 的 idx3 占 78.7%，上屏整件白大褂消失。修法 `convert_palettes.fill_interior_white` 洪泛填充——**「内部」必须定义为与画布边界不连通**，局部判断对凹形轮廓误判；复用那个函数不要自己写洪泛。
3. **门禁断言输出性质不断言过程**：不写「调用了 flood_fill」，写「转换后图像中不该透背景的区域像素≠背景色」——「守护绑在函数调用上，新加的 fetch_oak.py 直接绕过」这个真实复发就是教训。

### 8.5 溯源文化

`assets/eye_regions.json`（眼睛区域）、`pokemon_art_sources.json` 等把每个素材的上游来源坐标记成 JSON；`battle_fx_sources.json` 记 GSC OAM 图元的原作出处。数值同理：**数值不手填，从 pret 反汇编取**（`data/trainers/parties.asm`、`data/pokemon_moves/gold_silver.json`）。

---

## 9. Web 同源预览

> 「改页面观感先在 web 上定稿，再让固件复刻」——这是本项目最重要的开发效率设施。

### 9.1 两代预览的演进

1. **JS 参考稿**（`tools/inspector/template.html` + `gen_pages.py`）：sim/ 剧本驱动的 JS/Canvas 模拟器，八页按键驱动逐帧步进。问题：坐标/条宽与固件各写一遍，「颜色能 1:1 是因为有 screen.h 单一来源，布局不能是因为坐标各写一遍」（D28 实证）。
2. **固件同源预览**（最终形态，docs/12-firmware-preview.md）：**直接在浏览器运行固件的页面绘制代码**，浏览器只接收屏幕字节。

### 9.2 实际调用链（不是 Emscripten，不是手写 JS 绑定）

```
浏览器 firmware.html（页面选择/A B C 键/时钟步进）
  → HTTP POST /api/firmware {action: key|tick|page|reset|move_preview|audio|check}
  → 本地 server.py（ThreadingHTTPServer，仅 127.0.0.1，CSRF 检查 Origin）
  → 独立 native 进程（stdin/stdout 行协议）
  → 【真实固件代码】nav.c / play_*.c / battle*.c / assets.c / render.c / screen.c
  → LCD transport 替身接收实际 RGB565BE 字节
  → 浏览器 firmware.js 解码 + Canvas.putImageData
```

**关键设计：约 50 个生产 .c 原样编译到桌面**（`tools/inspector/native.py:20-28` 的 SOURCES 列表含全部 play_* 与逻辑层），用本机 `cc -std=gnu11 -O2 -DHOST_BUILD=1`，**不用 Emscripten/WASM**。资产用一段汇编 `.incbin` 生成 `_binary_*_start/end` 符号（native.py:56-61），与固件 CMake EMBED_FILES 的符号名完全一致。

**构建缓存按内容寻址**：对全部源码+头文件+host 夹具+7 个资产做 SHA-256，取前 20 位作版本号，缓存在 `.build/firmware-preview/<hash>/preview`；编译期间指纹变化会报错重载。「每个浏览器会话独立状态；右下方显示当前构建编号」。

**帧协议**：`FRAME <page> <ms> <mismatch>\n` + 240×320×2=153600 字节 RGB565BE 二进制；音频 `AUDIO <samples>` + PCM；状态 `STATE {json}`。HTTP 响应头带 `X-Preview-Page/Ms/Mismatch/Version/Music/Animation-Frames/Screen-Off`。

### 9.3 宿主夹具（tools/inspector/host/）——隔离 Wi-Fi/NVS/按键的正确做法

`runtime.c`（1041 行）的定位写得很清楚：「Page/render/asset/logic code is compiled from firmware/main **unchanged**. WiFi, NVS, audio and wall clock are **fixture services**; this is a rendering/interaction preview, not an emulation of those devices.」要点：

- **假头文件**：host/include/ 提供 `esp_log.h / esp_err.h / lvgl.h / esp_timer.h / bsp_display.h / esp_lcd_panel_*.h` 的最小替身。
- **lv_timer 模拟器**：自建 16 个定时器槽 + `advance(ms)` 按到期序精确回调——「暂停/单步 60ms」就靠它。
- **LCD 替身保真 DMA 语义**（这是精髓，runtime.c:73-96）：`esp_lcd_panel_draw_bitmap` 只**保留指针**（`pending_pixels`），直到生产 screen.c 发出 DMA 屏障（`esp_lcd_panel_io_tx_param`，即那个 NOP 同步点）才 memcpy 进全屏缓冲——**逼着被测代码遵守与真机相同的同步协议**；未同步的传输直接 `exit(3)`。`screen_host.c` 总共 6 行：`#include "../../../firmware/main/screen.c"`——生产 screen.c 整个编进预览，只换 LCD 传输。
- **确定性世界状态**：world/save 全套 fixture（可控的 `save_failure_after`/`save_delay_ms` 故障注入、starter 保存失败注入）；`boot <page> <pet> <level> <wild> <rarity> <seed> <shiny> <names> <team>` 一行参数构造场景。「测试场景不会读写硬件存档」。
- 全屏 150KB 缓冲只存在于开发机——「没有给 ESP32 增加全屏缓冲」。

### 9.4 会话与服务安全（server.py）

- 只绑 127.0.0.1；POST 校验 Host 是 localhost 且 Origin 同源（403 否则）。
- 会话 = `secrets.token_hex(16)`，LRU 上限 8 个渲染进程、30 分钟过期回收。
- 所有整数参数白名单校验（`integer(data, name, default, lo, hi)`）；页面枚举 P0-P6/P9-P16。
- `Cache-Control: no-store`；音频 PCM 直接 octet-stream（采样率 22050 放响应头）。

### 9.5 验收与证据

- `verify_firmware_preview.py`：跑八页+真实导航/按键/定时器回调，**局部刷新保留的画面 vs 同一时刻整页重绘**逐像素比对（848 次），覆盖对战/捕获/转场；负向对照「删除 DMA 屏障的副本必须失败」「删除宠物所在脏带绘制必须检出残影」。
- `reports/browser-firmware-parity-2026-09-07.json`：15 帧（7 页+8 个金银对战状态）浏览器 Canvas RGBA SHA-256 与独立 native 进程完全一致；320/835/1280px 窗口保持整数缩放无横向溢出。
- 另用独立原资产解码核对 151 只敌方精灵 2,355,600 个 RGB565 像素一致。
- RGB565 解码器本身对全部 65,536 色值核对。
- **诚实的边界声明**（文档原文级别）：预览不能代替真机验收 Wi-Fi 扫描、NVS 掉电、真实 LCD 刷新耗时/DMA 中断/色彩校准；P4 读时钟所以比对用最后一次推屏时间防误报。

### 9.6 moves.html 招式验收页

`tools/inspector/moves.html + moves.js + move-catalog.json`：单招逐帧验收（move_preview 命令带 move_id/side/frame/mode），99 招全部有表现、46 个 ID 定向映射、35 张 GSC OAM 图元 3372B。验收「敌方上移后的构图 45,984 个相位案例 + 八项负向检查」。

### 9.7 dungeon 预览

`tools/inspector/dungeon.py + dungeon.html/js`：/api/dungeon 独立端点跑秘境（`dungeon.RUNS` 进程池）。

---

## 10. 存档系统与 USB 备份

### 10.1 为什么 NVS 而不是自制双 buffer（save.h:3-20 完整论证）

S6 设计过「A/B 双槽 + CRC32 + 单调序号」，理由成立（任何时刻掉电至少一份完好）——**但 NVS 已提供同样保证**（每条记录带 CRC32、页级状态机、未完成写入在挂载时丢弃）。双 buffer 要自己做「先数据后 CRC」顺序、擦除块边界、磨损均衡，**三件事写错的表现都是「拔电偶尔丢档」——概率性、极难复现**。用 NVS 把它们交给被大量产品验证的实现。代价（字节布局与 sim/state.py 不一致）可接受——PC 侧不读设备存档。

分区：用现成 `nvs`（0x9000, 0x6000=24KB），不新增分区，上游三条契约不受影响。

### 10.2 布局与键

- 命名空间 `"pokewalk"`，整块 blob 键 `"state"`，独立 u8 键 `"opening"`（开场只播一次的单调标记——**独立键让页面无需写整块世界存档**；`save_write` 传 false 时绝不擦掉已写入的 true）。
- 写入三步：`nvs_set_blob` → （若 opening_seen）`nvs_set_u8` → **`nvs_commit` 不能省**（set 只写缓存不 commit 拔电就丢，而返回值是成功的）。
- 写前校验全部子结构有效（inventory/trainer/enc_refresh/exploration/rest_clock/dungeon），无效拒写。

### 10.3 前缀保持式版本链（save.h 最值得学的模式）

`SAVE_VERSION 17`，`save_t` 是嵌套 union 洋葱：

```c
typedef struct {
    union { save_v16_t v16; struct { ...v16 字段平铺... }; };
    rest_clock_t rest_clock;
} save_t;   // V17
_Static_assert(offsetof(save_t,rest_clock)==sizeof(save_v16_t),"V17 preserves V16 prefix");
_Static_assert(sizeof(save_v5_t)==2296, "legacy save layout changed");
_Static_assert(offsetof(save_t, inventory)==sizeof(save_v5_t), "V6 must preserve the complete V5 prefix");
// ... 每个版本一对"大小不变 + 前缀保持"断言
```

规则：
- **加字段 = 新版本 struct = 旧 struct 打头**，迁移变成「有界前缀读」。
- **每级 `_Static_assert` 锁死大小与偏移**——「存档版本 +1 时改了旧字段，编译不过」。
- `save_read_status()` 按 blob 长度识别版本（len == sizeof(save_vN_t)），逐级迁移（V5 补 inventory、V7/8 重排 trainer stride、V9→10 补 refresh、V10→11 补 exploration、V12 物理槽位化、V13→14 复用探索结构未用字节存 bitmask——**存档大小不变的新字段优先复用预留字节**）。
- 四态结果：`OK / EMPTY / ERROR / MIGRATED`——「只有 EMPTY 允许新游戏写入；ERROR 必须保留原 NVS 字节」（初始化失败不是擦玩家存档的许可）。
- 防御性细节：读入的 `_Bool` 字段先 `memcmp` 与 true/false 的标准表示比对再取值（原始 blob 可能含非法 bool 表示，直接读是 UB）；V5-V11 物种索引仓 `party[2+(PARTY_MAX+i)*MON_BYTES]` 校验 id 一致性。

### 10.4 事务性技巧（world.c 接口注释即规范）

全部「产出+凭证消耗」合并进**同一个临界区 + 同一次落盘**：

- `world_capture_uid(uid, mon)` — 「收容、点亮图鉴、按 uid 出队在一个临界区完成，随后立即把三者存档」。（历史缺陷 ⑫：顺序曾是 receive→mark_caught(立刻写NVS)→take_uid，中间掉电可重复捕获同一条。）
- `world_battle_loot_uid` — 「Once per won active session. Repeated calls return the same actual award without adding it again. A false return is retryable and publishes nothing.」
- `world_capture_ball_spend_uid` — 「Save one ball decrement and the throw's candidate session together」。
- `world_choose_starter` — 「Build and save a complete party+dex snapshot **before** publishing the leader. Failure leaves the pending choice unchanged; retry is safe.」
- `world_set_leader` — 「Save before publishing」；出队亲密度的 Q10 下取整「repeated switching cannot round up」。
- 运行时战斗会话（active/session）不进存档，重启即失，但「已开战」的 pending 移除**先落盘**，防掉电后已打过的目标复活。

### 10.5 自动存档分级（S18）

磨损计算翻转了假设：存档 ~1.2-3.7KB 占 1 个 4096B 扇区，NOR 10 万次擦写——每次遭遇存（28 次/天）能撑 **9.8 年**。「磨损不是约束，真正的约束是掉电丢多少进度」。分级判据不是「重要」而是「**丢了能不能再拿回来**」：捕获（尤其闪光）立即存、经验/照料 5 分钟节流存。`save_t.last_uptime_us` 存「上次开机微秒」（不是墙钟——没有 RTC 对时，只能知道上次跑了多久）。

### 10.6 USB 存档备份/导入（usb_backup.c + save-manager）

**设备侧协议**（`!PWBACKUP` 行协议，独占整行，畸形/超长不下落到旧按键注入）：

```
!PWBACKUP HELLO <32hex session>       → READY <session> <device_id>
!PWBACKUP BEGIN <session> <req_id> <mac> <fw_sha256> <schema> 0x9000 <bytes> <crc32>
!PWBACKUP DATA <req_id> <offset> <256hex=128B>   （tick 里分块发）
!PWBACKUP END <req_id> <crc32>        → 等宿主 ACK（文件关闭且回读校验后）
导入：OFFER 校验 mac/固件/schema/长度/crc → 设备确认（默认选中「取消」，B 选覆盖）→ 
      先导出当前存档做预备份 → RDATA 128B 分块上行 → RFIN 校验 CRC → STAGED → 重启应用
```

- 会话令牌 + 15 秒心跳失联 + 60 秒无确认超时；每次 ACK 匹配会话与请求 id。
- **导入安全**：64KB `save_restore` 分区（0x360000）暂存新旧镜像，提交标记最后写入；重启时在 NVS 初始化前安装并校验，失败回滚旧镜像；断电后重试未完成的已提交导入；校验失败且无法回滚时**停止启动游戏**避免继续写坏存档。
- 首次升级此功能必须**同时更新 0x8000 分区表和 0x10000 应用**，不能只写应用；不要擦整片 flash。

**宿主侧**（tools/save-manager/）：
- GitHub Pages（`https://Luobata.github.io/ESP32-PokemonGo/`，workflow `save-manager-pages.yml` 自动发布）——**Web Serial** 直接读写设备（桌面 Chrome/Edge，HTTPS 页面授权 USB + 目录授权），存档只写用户选择的本地目录，**不上传任何存档**。
- `.pksave` = 版本 1 JSON：设备标识、固件 ELF SHA-256 构建标识、存档版本、NVS 布局、CRC32、SHA-256、base64 分区镜像。**兼容边界是备份协议和分区布局而不是游戏版本号**——游戏 save_version 是原样保留的元数据（接受 1-65535），「后续游戏新增字段无需升级网页工具」。
- CLI `restore.py --restore --confirm-device <id>`：显式匹配设备 ID、写前自动备份当前 NVS、写后回读逐字节核对。
- 验证：`verify_usb_backup.py / verify_usb_import.py / verify_usb_backup_checkpoint.py`；故障测试覆盖暂存与恢复共 **871 个 I/O 中断位置**（含部分写入/擦除）。

---

## 11. 音频系统

### 11.1 全合成、零 PCM

- `sound_mixer_t`（sound_mixer.h）：一个结构混三路——`music_player_t`（**4 声部** chiptune：note/noise/phase/increment/duty）、`cry_player_t`（**3 声部** GB 风振荡器复刻金银叫声：envelope/duty/lfsr/duty_frame，「Original Gold note programs/parameters, mono GB-style oscillators (not a cycle-exact APU)」）、SFX 效果器（含一阶 DC 滤波 previous_input/output）。
- 18 首音乐枚举（music.h）：OAK/LAB/HOME/CENTER/ROUTE/GYM/WILD/TRAINER/LEADER/CHAMPION + 三种胜利 + EVOLUTION/ENCOUNTER/CAUGHT/EVOLVED。音符数据从 `assets/music_sources.json`（金银轨道提取：extract_gold_animation_tracks.py / compile_gold_animation_tracks.py）编译成 `music_assets.h`。
- 采样率 22050 Hz（server.py `X-Audio-Sample-Rate: 22050`）。

### 11.2 播放任务（sfx.c，全部硬约束来自实测）

```c
#define SFX_CHUNK_SAMPLES 480
s_queue = xQueueCreate(8, sizeof(request_t));
xTaskCreate(sfx_task, "sound", 4096, NULL, 5, NULL);   // 优先级 5 高于 LVGL/world 的 4
```

- **为什么独立任务**：codec 写入最多阻塞 1 秒（`DEFAULT_WAIT_TIMEOUT=1000`），DMA 只有 65ms 深度——在 LVGL 任务或按键回调播必掉帧。
- **为什么优先级 5**：高于 LVGL 与 world 的 4。
- **开机预热** `set_format`：首次调用必触发 DMA free+alloc（buf_size 960→480），预热时 heap 75296B、`MALLOC_CAP_DMA` 最大连续块 59392B（I2S 需 6×480=2880B **连续**块——**最大连续块比总量重要**，`heap_caps_get_largest_free_block` 与总量一起打日志）。
- **队列纪律**：满则丢弃新请求不卡调用方；任务循环先**排空队列只留最新**（「Latest effect takes priority over stale menu clicks; queue cannot build an audio backlog」）。
- **懒开关 codec**：静音/熄屏时 `bsp_audio_suspend()`，有事件才 init+set_format；`xQueuePeek` 阻塞等事件且不丢「检查与阻塞之间送达」的事件。
- `sfx_encounter` 用 atomic CAS 维持最高级别警报（shiny>高稀有>普通）。
- `CONFIG_POKEWALK_SILENT_BOOT` 静默构建：所有接口编译成空，不建队列不碰 codec。
- 静音持久化 + 音量 10 档设置（audio_settings.c）；`bsp_audio_boot_quiet()` 是 app_main 第一件事（接了功放的设备开机不炸音）。

### 11.3 音频验收的方法论（值得单独记）

S13 曾判 PASS（欠载 0/写错误 0/样本数与 sim 吻合）而**用户听不到任何声音**——`bsp_audio_set_volume` 全项目零调用，codec 停在硬件默认值。「那些指标量的是『样本写出去了』，不是『有声音出来』」。修法是补**能证明可听**的手段：dbg.c 的 `r` 命令用**物理麦克风回采**（ES8311 全双工 `bsp_audio_read`）——静音 RMS 51 → 播放 RMS 3991（**78×**），峰值 int16 上限 37~45% 未触顶 clip 0。「下次判音频看这个，不看欠载计数。」Web 侧另有 WebAudio 现场合成（`firmware-audio.js` + `/api/firmware audio`），浏览器里能听「本该是什么声音」。

---

## 12. Wi-Fi 感知玩法

（`docs/02-sensing.md` 350+ 行 + `firmware/main/sensing.c/h` + `sim/sensing.py`）

### 12.1 核心设计观

- **扫描不需要连接 Wi-Fi**：`esp_wifi_scan_start()` 在 STA 模式下工作，不关联不要密码；隐藏 SSID 照样可用（beacon 里 SSID 空但 BSSID 在，指纹靠 BSSID）。整套方案全程不联网成立。
- **Wi-Fi 是上下文开关，不是内容生产者**：「让时间生产内容，让空间调制内容」——任何基于「见到新 BSSID」的指标都会衰减到零（第一周发现完，第五周变砖），**任何会衰减到零的指标都不能作为长期驱动力**。
- 可再生信号表：地点身份/驻留时长/状态转换/同地点昼夜指纹（三个地点白送变六到九个上下文）/久别重逢/RSSI 微抖动。
- 降级阶梯：A 地点集合（8 槽 LRU 512B）→ B 单比特移动判定（不需要数据库，「通勤中」本身就是一个 biome）→ C 驻留累积 → D 昼夜上下文。最坏退化成纯 Tamagotchi「也没问题，它照样成立」。

### 12.2 匹配算法（Q10 定点）

```
w(rssi) = clamp((rssi + 100) / 60, 0, 1)    // -100dBm→0, -40dBm→1（SENS_RSSI_FLOOR/CEIL）
sig = { (hash32(bssid), w) } 取 top-8        // SENS_TOP_N 8，48 字节定长
score(A,B) = Σ min(wA,wB) / Σ max(wA,wB)     // 加权 Jaccard
match: score ≥ 0.35（358/1024）→ 命中该槽
move:  score < 0.40（410/1024）
迟滞：状态切换需连续 2 次一致（SENS_HYSTERESIS 2）
```

定点论证（sensing.h:8-18）：Q10 与浮点在 1061 次真实扫描上判定**零翻转**、相似度最大误差 0.00115；Q10 的选择是「TOP_N=8 时相似度分母最大 16384，u16 装得下」。

### 12.3 滑动窗口（真实数据驱动的修正——本节是全项目最好的「实测推翻设计」案例）

单帧比对在家静坐 20 分钟实测：状态转换 11 次（应 <2 次/小时）、30% 误判移动。根因：28 个 AP 只有 9 个出现率 ≥59%，另 19 个 ≤37% 且几乎全在 -80dBm 以下，单次 top-8 被噪声 AP 主导。**静态过滤全部无效**（RSSI≥-70 / top6+阈值都不行）——「问题不在阈值，在逐次比对本身」。修法：4 帧滑动窗口（`SENS_SMOOTH_WINDOW 4`，每帧存 `SENS_FRAME_APS 40` 个去重 AP 的 hash+weight），`weight(h) = 出现率 × 平均rssi_weight`，用窗口指纹算距离。FRAME_APS=40 的依据：实测单次最多 31 个 2.4G AP（办公室 65 是含 5G 的总数）；窗口 4×40×6=960B。**第一版存距离值取平均是完全不同的东西**（正确做法是用出现率重建指纹再算距离）。

环形缓冲遍历序也踩过坑：按数组下标扫环绕后 ≠ 时间序，与 Python deque 不同序会让并列权重下 take_top 选中不同 AP——修成按 `win_head` 旧→新遍历。

### 12.4 两级扫描（功耗）

被动扫描每信道 >100ms beacon 间隔，13 信道全扫 **1.4 秒射频全开**；30 秒一次占空比 4.7%。优化洞察：「**做指纹不需要看全，需要看得一致**」——驻留时只扫 1/6/11 三主信道（0.3 秒）问「我还在原地吗」；锚点丢失或每小时一次才全扫问「我在哪」。诚实张力：「AP 少时强制全扫，只在 AP 密集时才敢裁剪——城里省电，家里不省」。实现上还有自适应占空比（驻留退 5 分钟，churn 加速 30 秒；熄屏静止 30→60 秒）。

### 12.5 地点槽与 biome

`sens_place_t`（24B）：pid + 48B 指纹 + first/last_seen + dwell 累计驻留 + biome（**首次分类后冻结**）。8 槽 LRU。`dwell_by_biome[5]` 分野外/住宅/办公/商业/交通累计。biome 判定的教训：`MIN_APS_FOR_BIOME=4` 保护使「野外」分支数学上不可达（穷举 n=0~8 验证过）——「分支存在 ≠ 会执行」；办公环境 17% 误判野外 → **稀疏扫描不可信**。

### 12.6 与固件的连接

world.c 任务扫完喂 `sens_feed()`，产出 `sens_result_t`（distance Q10/transient_aps/is_new_place/place_id/match_score）驱动：移动量累积（`PROGRESS_FULL_Q10 = 20*1024`，**数字来自真实数据**：通勤 26 分钟=9.3、家 1.2h=0.3、办公 8h=1.5，一天两趟约 18.6，取 20 满格让正常通勤日 93% 纯坐办公室 7.5%）、猎场遭遇、探索发现。`world_progress_from_motion()` 单独导出「与 PC 侧对账用，能在宿主上测」。

---

## 13. 构建 / 烧录 / 分区 / 增量更新

### 13.1 分区表（firmware/partitions.csv）

```
# Name,   Type, SubType, Offset,    Size,     Flags
nvs,      data, nvs,     0x9000,    0x6000,        # 游戏存档（24KB）
phy_init, data, phy,     0xf000,    0x1000,
factory,  app,  factory, 0x10000,   0x340000,      # 应用 3.25MB
cardid,   data, nvs,     0x356000,  0x4000,        # 上游契约不能动
save_restore, data, 0x40, 0x360000, 0x10000,       # USB 导入暂存 64KB（后加）
wifi_config, data, nvs,  0x370000,  0x6000,        # 配网凭据（后加）
recovery, app,  test,    0x700000,  0x100000,      # 恢复分区 1MB（上键5s进入，上游契约）
```

已知警告：recovery 1MB 装不下 1.45MB+ 应用（构建一直警告，未改——**三条 BLE 安装契约是强制的不动**）。

### 13.2 fw.sh（tools/device/fw.sh）——构建/烧录/备份三件事一个脚本

- **烧写前硬门禁：没有任何备份就拒绝烧**。理由写在脚本头：设备出厂可能是别人的玩法（sunny0826 离线图鉴 1025 只精灵 5.9MB 数据分区），覆盖就没了，重装要走小程序 BLE。「上游文档 STEP 05 写了『安装前保留可恢复版本』，这里把它变成硬门禁」。备份 = `esptool read_flash 0x0 0x800000`（整 8MB，几分钟）存 `.device-backup/flash-<时间戳>.bin`。
- **端口自动探测**：优先 pyserial 找 `vid==0x303A`（Espressif），退路 glob `/dev/cu.usbmodem*`——「usbmodem 编号会变（11301→1101），硬编码必踩坑」。
- **build 前自动重生成字库**（2 秒 PIL）：把「加一句中文日志引入新字」这类隐性失败吸收进构建；没装 PIL 跳过并提示。
- `restore FILE` 用 esptool `write_flash 0x0` 写回整片。
- `monitor` 用自家 monitor.py（解析硬件事实表）而非 idf.py monitor（独占终端）。

### 13.3 idf-env.sh——环境包装脚本

记着一个真实坑：pyenv 的 python3.12 自编译缺 lzma 模块解不开 IDF 工具链 .tar.xz，export.sh 失败。解法：PATH 前置系统目录让 python3 解析到 /usr/bin/python3（3.9.6 自带 lzma），homebrew 的 cmake/ninja 跟在后面。「对账脚本要编译 C，用系统 python3」这条选择已固化进脚本。

### 13.4 两种烧录形态

- **开发增量**：`idf.py -p PORT flash` 只写应用（0x10000 起），不动 bootloader/分区表/NVS——进度保留。社区版发布后设备「图鉴 90、伙伴 83、喷火龙 39 级 92023 经验保持」就是这么验的。
- **社区合并固件**：`tools/release/package_firmware.py` 用 `esptool merge_bin --flash_mode dio --flash_freq 80m --flash_size 8MB` 合成 `FoloToy-AI-Passport-full.bin` 烧 0x0（bootloader@0x0 + 分区表@0x8000 + 应用@0x10000），**不含真实 NVS/cardid/recovery 内容（NVS 空白填充）**，随后跑官方校验器。同时输出 `flash_args` 与 manifest（SHA-256、应用大小、烧录地址）。**注意**：合并固件会覆盖存档——旧设备要走「保留存档的组件更新流程」或先 USB 备份；save_restore 新分区要求同时更新分区表+应用。

### 13.5 串口工具家族（tools/device/）

- `monitor.py --info` — 一条命令看硬件事实（只要 pyserial 不用装 IDF）。
- `screenshot.py --out x.png --wait` — 收 base64 帧存 PNG。
- `walk.py --keys e,c,A,b,a,a --settle 26` — 串口注入按键走链路+逐页截图；小写=单击大写=双击 s=截图。**--settle 必须给足**：默认 2.5s 只覆盖 4 回合，战斗最坏 43.2s，验结算用 `--settle 46`。
- `decay.py --minutes 150` — 衰减速率实测。
- `soak.py` — RTC 漂移长跑。
- `collect.py` — NDJSON 扫描采集（data/raw 三份真实数据 home/commute/office 的来源）。
- dbg.c 串口命令（CONFIG_POKEWALK_DEBUG_KEYS 默认开，发布可编译掉）：按键注入、`p <id>` 主动触发音效、`r` 麦克风回采 RMS、战斗种子覆盖、按钮 ADC 探针（`@@BUTTON_PROBE` 每 100ms 打 adc_mv+GPIO_IN，被动只读）。

---

## 14. 社区发布流程

（`docs/release/community.md` + `docs/release/submission.json` + `reports/evidence/community-*`）

### 14.1 平台与项目

- 平台：AI Passport 社区 `https://ai-passport.folotoy.cn`，创作者工作台 `https://ai-passport.folotoy.cn/account/?project=234`。
- 项目 **234 / pokewalk**，分类 **games（游戏与互动）**。版本序列 378→400→401→404→607→613→624→631→804→815→1074（每版 pending→审核通过→下一版）。

### 14.2 完整 checklist（照抄可用）

1. **产物**：`source tools/device/idf-env.sh && tools/device/fw.sh build && python tools/release/package_firmware.py` → `release/community/FoloToy-AI-Passport-full.bin`（0x0 烧录）+ manifest.json（SHA-256/大小/flash_address/protected_layout_pass）。
2. **官方校验**：package 脚本内嵌跑 `verify_firmware_upstream.py`（官方原样校验器，检查组件一致性、分区 MD5、应用容量、恢复入口、保护区）+ `verify_firmware.py`。「证明产物结构通过官方检查，不等同于已通过社区 BLE 安装/更新实测」——这句边界声明保留在文档里。
3. **封面**：480×640 **3:4 竖版**（JPEG/PNG/WebP ≤10MiB）。本项目的经验：第一次封面被截图糊弄，换成「清晰的原始战斗帧，双方精灵完整可见，无遮挡」的录屏截帧（78.5 秒处）。
4. **文案**：`submission.json` 双语标题与介绍（接口 **4000 字符上限**，超了要压缩，完整版留仓库）；「版本限制写明，不能宣传为金银全招式复刻」——不夸大。
5. **发布**：官方 publisher 工作流 zip（`https://ai-passport.folotoy.cn/skills/folotoy-ai-passport-publisher.zip`）→ 社区登录「发布新玩法」→「✦ 用 AI 辅助发布」→ **确认连接发布助手授权码（授权入口默认折叠，无需把密码交给助手）** → 先跑 `whoami` 与 `projects` 核对授权和项目号 → **如已有草稿或待审版本，停止并报告冲突** → 选固件/封面/源码 HTTPS 地址/submission.json 双语介绍 → 预览后上传。
6. **更新已有版本**：换封面/改分类用官网「撤回并修改」再编辑重提；**不要另建项目**（分类被接口重置成 developer 是官方 bug，去网页改回 games）。
7. **回执存档**：每次发布把提交回执、固件 SHA-256、CI 链接、烧录验证存进 `reports/evidence/<topic>/`（community-release.json / device-flash.json / cover-update.json...）。
8. **发布前设备验证**：发布构建重新烧录当前设备、三段写入校验、启动摘要匹配、**玩家进度保持**（图鉴 N、伙伴总数、等级经验逐项核对）、静音状态保持。
9. 归档到上游 `plays/` 是单独的双语文本 PR，不传固件或封面。

### 14.3 每版发布说明文档结构

`reports/release-notes-<date>-<topic>.md` + `docs/release/<date>/changelog.zh.md + instructions.zh.md`（社区独立的「使用说明」「更新日志」字段，**增量日志只含本轮变化**，历史说明留链接）。

---

## 15. 验证文化与门禁体系

### 15.1 数量与形态

`tools/pipeline/` 共 173 个脚本，其中 **135 个 verify_***。核心十几个（00-handoff.md 一条命令串起）：

```
verify_battle      791 组   S2·S3 确定性逐值 + 三组难度梯度
verify_encounter   961 组   S1·S5·S8，含队列全量内容比对
verify_nurture    3201 拍   S4 衰减 + 三动作 26 步 + 动作×衰减混合
verify_exp         651 组   经验曲线/升级边界/cap/进度（0..255 全域 + 每级边界三点 + 溢出邻域）
verify_party       259 组   Mon 布局/better/receive/200 连捕/序列化字节数
verify_opening     527 帧   开场状态机逐帧五元组 + A 两语义 + C 跳过
verify_transitions 134400格 八种转场逐格 + 32 组选型（含 uint8 溢出边界）
verify_audio     76985样本 11 条音效逐样本 + 2442 段分段一致性 + MIDI 全枚举
verify_ui          7 素材   ui.bin 结构/条目/逐像素/非空白
verify_layout    7 个页面   绘制坐标是否跨 80px 横带（BUG-1 的守护，静态扫描）
verify_sensing    ⚠️ 已知差异  C 定点 vs Py 浮点的已知数值路径差异（state/trans=0）
verify_sim_pages            页面模拟器与 sim/ 逐值一致（含新鲜度断言：index.html 是生成物过期了浏览器看不出）
verify_firmware_preview    848 次局部/整页对照 + DMA 屏障负向对照
inventory_assets           资产断言，退出码当 CI 门禁
```

后期每加一个系统配一个：verify_dungeon_*、verify_exploration_*、verify_trainer_campaign、verify_usb_backup、verify_display_ownership、verify_directional_controls、verify_stalemate、verify_gen2_damage……（135 个覆盖每个功能）。

### 15.2 门禁脚本的标准骨架（verify_exp.py 为范本）

- **宿主驱动用真的 .c**：内嵌 C 主程序 `#include "exp.c"`（不是替身）——「替身一旦与真的不同，对账全绿而固件全错。这是整个门禁体系里最容易犯且后果最大的错」。
- `cc -O1 -Wall -Wextra -Werror`——「宿主编译比目标编译更严，顺手抓隐式转换与未初始化」。
- 驱动是**行协议**（stdin 命令 + 参数，stdout 值 + fflush），Python 侧比对。
- **逐值而非抽样**：「嵌入式的数值缺陷典型形态是『差一点但一直差』——除数抄成 3、截断方向反了，分布上只差一两个百分点，抽样统计过得去，逐值立刻红」。
- 覆盖清单：常量逐一 / 定义域全量（uint8 就 0..255）/ 每个分段边界三点 / 溢出邻域钉钳制值 / **锚点**（人工算过的具体值——逐值对账验「两边一致」，两边同源同错它全绿，锚点是唯一能挡住这个的）。
- 用法支持 `python3 verify_x.py <别的.c路径>` 专门给反向验证传副本。

### 15.3 门禁三律（10-framework.md 第三章，全部有实证）

1. **反向验证**：故意改坏被测代码必须退出码非 0。「恒绿的门禁比没有门禁更坏——它让后面每轮都误以为验过了」。
2. **二阶陷阱**：反向验证先**断言改坏生效**（`diff` 非空）再看退出码——真实踩过 sed 模式没命中、副本没变、门禁照常绿，「验证门禁的手段自己恒绿了」。
3. **被测文件缺失时也必须红**（`ALLOW_MISSING=1` 可显式容忍未开工期）——「『还没做』和『做对了』不能是同一个信号」。

### 15.4 verify_layout.py：静态扫描三律

呈现层门禁（补「七个 play_*.c 共 1913 行曾零覆盖」的最大空洞）：解析 #define/函数宏/常量数组/局部变量/for 循环域，对绘制调用坐标求值检查横带不变量。三条纪律：
1. 无法静态求值的**显式报 SKIP 并打印**，绝不静默通过（「被检查的 bug 的成因就是越界像素静默丢弃——检查器自己不能犯同一个错」）。
2. 退出码只由确定违规决定，SKIP 不计入失败（否则永远红然后被忽略，等于没有）。
3. **豁免必须带理由、绑定行号、逐条打印**（代码一挪豁免就失效，有意如此）；「绝不允许为了让门禁变绿而加豁免」。

它上线时抓到 3 处真违规，其中一处正是**验收证据本身所在的那行文字**（「经验 +N」渲染行跨带）——「验收证据可能落在缺陷上而不自知：dump 路径一次性重画四条带不存在频率差，截图看不出来」。

### 15.5 观测手段五问（10-framework.md 第四章，最重要的一章）

拿到一个验证手段，按顺序问：
- **Q1** 目标故障现在就发生，我的输出会变吗？（观测点必须在故障点**之后**——截图取字节序交换前就永远看不见字节序错）
- **Q2** 我的观测动作会改变被观测状态吗？（复位测衰减=每次测初始值）
- **Q3** 采样率/分辨力够吗？（定量算：帧率/传输字节/波特率/超时，不估）
- **Q4** 故障表现形式在我的通道里存在吗？（像素问题不指望日志、时序问题不指望单帧截图）
- **Q5** 如果它永远不报警，我怎么知道是「没问题」还是「坏了」？（= 反向验证）

三种失效形态各有实证：观测绕开出错环节（DMA 竞争紫屏但截图取交换前数据）/观测扰动状态/原理上无分辨力（跨帧中间态四条通道全不可见）。结论：「观测覆盖不到的地方，不是逻辑写错」——项目全部历史缺陷的根因都是同一个形状。

### 15.6 reports/ 验证报告模式

`reports/` 按主题命名（`<topic>-<date>.md`），`reports/evidence/<topic>/` 存截图/JSON 回执/探针脚本。每份验证报告末尾附表：逐条标注「我的手段能不能看见这条结论的反例」，三档**可信 / 量级估计 / 未验证**。「『没测到』与『没发生』必须分开写」——『实测未发现冲突』= 我跑了观测到正常；『静态分析未发现阻塞性冲突』= 我没跑我推的。

---

## 16. 工程方法论

（docs/00-handoff.md「踩过的坑」九条 + 10-framework.md + STATUS.md——这是最值得提炼进 skill 的部分）

### 16.1 三条原则（一切做法的出发点）

1. **能在宿主上验的，绝不留到设备上验**（设备一轮几十秒观测弱；宿主几百毫秒工具全）。
2. **能在编译期挡住的，绝不留到运行期**（运行期在设备上，而设备上你看不见）。
3. **观测手段必须能看见目标故障——这一条要显式验证**（「测了没问题」和「测不到问题」在报告里长得一模一样）。

### 16.2 什么项目值得这么做（避免过度工程）

不值得：几百行的点灯/MQTT 项目、一次性 demo。值得的判据（满足两条以上）：有可对账的数值逻辑（状态机/曲线/判定表/序列化——「差一点不会崩只会一直错」）；有呈现层（故障形态是「不对」而非「崩」）；生命周期超几周或要交接；有可作基准的参考实现。

成本量化：单个门禁 100~400 行 Python+内嵌 C；反向验证每个约 10 分钟；**观测通道（屏幕转储+按键注入）一次性各约 150 行，是收益最高的一笔**；布局静态扫描约 200 行抓到 3 处真缺陷；编译期断言每处 1 行几乎免费。

**不要做的**：不为「体验够不够好」造门禁（产品判断，自动化它=主观规则堆积）；没有基准不推逐值对账；不为凑覆盖率给平台层写门禁（mock 与真实平台的差异本身就是缺陷来源）。

### 16.3 对账门禁的固有上限

**对账门禁的上限是基准的完整度**——基准侧只是 bool 无实现，目标侧同样没实现，两边一致全绿，用户看到功能缺失。对策：建「基准缺口清单」（有标志位无实现的地方登记为移植前必须决定做不做的待决项）+ **死字段检测**（扫「有赋值无读取」，白名单 `// dead-ok: 理由`）。

### 16.4 handoff 九坑（每条都是真实事故）

1. **验证手段与被验对象共用同一条会坏的路径**（栽了四次：模拟器 JS 死了拿它自测表验收；1× 渲染漏目标尺寸整轮证据图摄于残屏且穿过三个检查；GSC 参照图用同一套渲染代码重建两边一起错；门禁日志失败也打 ✓）。
2. **报缺陷前先怀疑自己的脚本**（大面积不符更可能是解析姿势错：moves.bin 头 16B 按 14B 切得到 move_id=834；测量探针翻车九次全是探针错）。
3. **按模型上限算不是按样本算**（只查 2 字招名漏了 5 字「尖刺加农炮」256px 溢出；真机 miss 测试用 seed=31 但六尾只会火花命中率 100%，什么种子都不会 miss）。
4. **布局改动先列所有元素区间表再改**（逐个挪把重叠推给下一个：修消息行压到名牌）。
5. **静默失败比崩溃难查**（canvas fillStyle 收到 undefined 不报错不画；gen_pages.py 长期 exit 1 失败被当常态）。
6. **观测手段坏了 ≠ 被观测对象坏了**（投递失败被当成对方没干活，四分钟后人家交了完整 result；判断死活的独立证据当时全有，先下结论才找证据）。
7. **同一规则三处各写一遍就会有三个值**（HP 段位：asm 像素判据 / web >21% / 契约 >20%。修法照 asm 的形状写，条宽改了不用重推）。
8. **字库靠注释兜底过**（「攻」进字库靠 battle.c 注释里的「一次攻击」被源码扫描捞到，删注释字就没了——收进 strings.py 单一来源）。
9. **唤醒信号不是连通性探针**（裸 nudge 让对方空跑 14 分钟只读核查然后发 blocker 说「这个任务不存在」——它是对的）。

### 16.5 多角色/多 agent 契约纪律（10-framework.md 第八章）

- **权限只从契约来，消息不授权**：收到「请改 X」而 X 不在写允许清单，正确做法是拒绝并要求更新契约（三次正确拒绝越权的实证）。
- **一路一个写者**：同路径同轮只一个写者，「谁改坏的永远可追」。
- **契约错误由执行者拦下，且这是设计的一部分**：协调者五次实质错误（语义/编号/漏同类/归因/类型装不下）全部在动手前被执行者或门禁拦下。「契约的价值不只是分工，而是**把假设写下来供人反驳**」。
- **执行者要复核协调者给的前提**：「可用堆 187KB」实际引用了另一个固件的数字（真值 231KB）——契约里的关键数字动手前自己复核并把复核方式写进报告。

### 16.6 新项目落地顺序（10-framework.md 附录 A）

1. 先定分层（哪些纯逻辑不碰平台头）——最便宜且最不可事后补。
2. 建两条观测通道（屏幕转储+输入注入）——之后每轮验证自助完成。
3. 第一个纯模块+第一个门禁+反向验证——走通一遍，后面照抄。
4. 编译期断言逐条加上。
5. 扫一遍规范性注释（本项目 56 文件 85 条，30% 可机械检查），按价值逐步变门禁。
6. 资源水位与状态轨迹日志在第一次偶发问题**之前**加好。

### 16.7 反面清单（10-framework.md 第十章，照抄进任何 skill）

| 看着对其实无效 | 正确做法 |
|---|---|
| 门禁全绿就放心 | 反向验证 |
| sed 改坏做反向验证 | 先 diff 断言改坏生效 |
| 门禁断言「调用了某函数」 | 断言输出性质 |
| 不变量写在注释里 | _Static_assert 或门禁 |
| 截图当动画/时序证据 | 状态轨迹日志+帧号 |
| 截图当「用对了数据源」的证据 | 度量 bbox 等可量化特征 |
| 自检通过当「画对了」 | 渲染改动必须看屏幕 |
| 越界静默丢弃 | 计数器 |
| 拿不到锁就读旧值（无影子副本） | 独立影子副本 |
| 慢操作放临界区 | 挪锁外 |
| 正确性依赖外部锁但不写明 | 头文件写线程契约 |
| 对账基准本身缺内容 | 基准缺口清单+死字段检测 |
| 手画素材凑数 | 同源同管线 |
| 相信上级给的数字 | 关键数字自己复核 |

---

## 17. 可复用技术清单速查表

**架构决策（12 条最重要的）**

1. 纯逻辑层 0 平台 include + 宿主 cc 编译真 .c 对账（§3.1）
2. 显式页面表 + enter/exit/key 三函数 + nav_ctx 单槽 uid 传参（§3.2-3.3）
3. world 独立任务 = 状态唯一所有者 + 值快照 + 双锁单向顺序（§3.4-3.6）
4. 240×80 横带渲染 37.5KB static + NOP tx_param 逼 DMA 同步（§4.1-4.3）
5. `SCREEN_ASSERT_WITHIN_BAND` 编译期布局断言 + 静态扫描三律（§4.4/§15.4）
6. screen_own_display：LVGL invalidation 关闭 + refresh timer 暂停（§4.5）
7. 确定性遭遇（crc32(bssid, hour)）→ 跨设备一致+可对账+免存刷新表（§6.1）
8. NVS 整块 blob + 前缀保持式版本洋葱 + 每级 _Static_assert（§10.1-10.3）
9. 产出与凭证消耗同临界区同落盘 + uid 幂等（§10.4）
10. 生产 C 原样编译到桌面（HOST_BUILD）跑 Web 同源预览，LCD 替身保真 DMA 屏障（§9）
11. 串口按键注入 + base64 截图 = 自助验证两半（§4.6/§13.5）
12. 资产 CMake EMBED_FILES 进 app 分区，头部自描述 + 自洽校验（§8.2）

**具体常量速查**

| 项 | 值 | 文件 |
|---|---|---|
| 横带 | SCREEN_BAND_H 80，4 条，缓冲 37.5KB | screen.h:17 |
| 字库 magic | "FNT1"，14B 头，升序 u16 索引 | render.c:39 |
| 物种表 magic | "GEN1"，16B 头，32B/记录 | assets.c:60 |
| 2bpp | 4px/字节高位左，色号 3 透明 | render.c:210 |
| 队伍 | mon_t 12B，PARTY_BYTES 1886 | party.h:8-9 |
| 遭遇队列 | 物理 16 槽/可见 5 条/条 16B | encounter.h:19-21 |
| 图鉴 | 4×19B 位图 | encounter.h:22 |
| NVS | 命名空间 "pokewalk" 键 "state"+"opening" | save.c:34-36 |
| 分区 | nvs@0x9000/24K，app@0x10000/3.25M，save_restore@0x360000/64K，recovery@0x700000/1M | partitions.csv |
| 感知 | Q10、top-8、0.35 匹配/0.40 移动、迟滞 2、LRU 8 槽、窗口 4×40AP、布隆 8192bit | sensing.h:27-46 |
| 音频 | 任务优先级 5 栈 4096、队列 8、块 480 样本、22050Hz | sfx.c:28-98 |
| 扫描 | 30s 间隔、MAX_APS 64、全扫 1.4s/三信道 0.3s | world.c:42-43 |
| 今日行程满格 | 20×1024 Q10（来自真实数据） | world.c:60 |
| 经验曲线 | floor(3n³/2)，Lv100=1,500,000 | exp.c/S32 |
| 熄屏手势 | C 长按全局拦截；B 长按=返回；BSP 抑制长按后的 click | nav.h:59-68 |
| 截图协议 | `@@SHOT 240 320 rgb565be 80` / `FAP_SCREENSHOT_V1 ... RGB565LE` | screen.c |
| 备份协议 | `!PWBACKUP`，128B hex 分块，15s 心跳/60s 超时，.pksave JSON v1 | usb_backup.c |
| 社区 | 项目 234/pokewalk，3:4 封面 ≤10MiB，合并 bin 烧 0x0，接口 4000 字符 | docs/release/ |

---

## 附：一句话总结

这个项目证明了：在 231KB 可用堆、三按键、无 GPS 无 IMU 的硬件上，靠**横带渲染 + 确定性玩法 + 纯逻辑分层 + 135 个宿主门禁 + 生产代码同源 Web 预览 + 前缀保持式存档 + 串口自助验证**这一整套工程体系，可以做出并持续发布（社区版本号迭代 10+ 次）一个完整的中文宝可梦 RPG。它的每一条规则背后都有一个真实缺陷做注脚——「注释是知识，不是守护」。
