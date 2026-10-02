# 路线 B：纯 C 自研渲染路线（横带软渲染 + 自建页面表）

> 蓝本：《PokeWalk》（`D:\aipassport-game\ESP32-PokemonGo-main`），ESP-IDF 5.5.3 + 官方 BSP（屏/键/音频）+ 自建渲染与导航。适用：像素风/精灵图/追求极致内存与帧率控制/大量自定义绘制。深度解剖见 `case-study-pokewalk.md`。

## 1. 为什么自研而不是 LVGL

- 整屏 RGB565 150KB = 可用堆 231KB 的 65%，LVGL 对象树 + 内部堆是额外开销；自研横带只要 37.5KB。
- 像素风的全部表现力在 sprite/调色板/整数倍缩放，LVGL 控件反而是负担。
- 自研渲染层极小（screen.c + render.c 数百行），且**同一份代码能在 PC 上原样编译跑 Web 预览**（见 §8）。

## 2. 横带渲染（核心内存决策）

```c
#define SCREEN_W 240
#define SCREEN_H 320
#define SCREEN_BAND_H 80                          // 240×80×2 = 37.5KB
#define SCREEN_BANDS (SCREEN_H / SCREEN_BAND_H)   // 4 条，80 整除 320 无余数
static uint16_t s_band[SCREEN_W * SCREEN_BAND_H]; // 唯一的帧缓冲（static！）
```

- 整屏 150KB static 会在**链接期**挤爆 dram0_0_seg（实测溢出 32624B 链接失败）；37.5KB 是安全值。
- 绘制接口只有 4 个：

```c
uint16_t *screen_band(void);                   // 直接写缓冲（sprite 快路径）
void screen_band_clear(uint16_t rgb565);
void screen_px(int x, int y, uint16_t c);      // 带内坐标，越界静默忽略（要可计数）
void screen_push_band(int band_y);             // 推屏（内部处理字节序）
```

- 页面绘制标准形状：

```c
static void draw_band(int band_y) {
    screen_band_clear(GAME_UI_BG);
    #define Y(v) ((v) - band_y)          // 整屏坐标 → 带内坐标
    render_text(12, Y(40), buf, GAME_UI_INK);
    #undef Y
    screen_push_band(band_y);
}
static void draw_all(void) { for (int y = 0; y < SCREEN_H; y += SCREEN_BAND_H) draw_band(y); }
```

## 3. DMA 异步竞争（本路线最深的坑，解法极妙）

**问题**：`esp_lcd_panel_draw_bitmap` 是异步的（排进 SPI 队列就返回）。四条带连着画：带 0 排队→立刻返回→带 1 重填 s_band→带 0 的 DMA 读到带 1 的像素。实测症状：紫/绿混杂、底部闪烁。

**三个解法的取舍**：
1. `on_color_trans_done` 回调？**不行**——回调槽已被 esp_lvgl_port 占用（与 LVGL 共用 panel_io），一注册就顶掉 LVGL 的，看门狗复位循环。*一个 panel_io 只有一个回调槽。*
2. 双缓冲？再要 37.5KB DRAM。
3. **最终方案（零内存开销）——用一次 tx_param 逼出同步点**。读 esp_lcd_panel_io_spi.c 源码发现：任何 polling 传输前，驱动会先把队列里在途传输收干净。所以发一条最便宜的命令 = "等前面画完"：

```c
static void wait_dma_done(void) {
    esp_lcd_panel_io_handle_t io = bsp_display_io();
    if (io) esp_lcd_panel_io_tx_param(io, 0x00, NULL, 0);   // NOP，几微秒
}
```

## 4. 与 LVGL 共存（用官方 BSP 时必读）

游戏不建 LVGL 对象直接画横带，但 LVGL 任务还在跑。坑：`lv_screen_load()` 空屏会**排队一次整屏刷新**，在游戏首帧后执行就用空背景盖掉游戏横带（部分重画的页面留下永久色条）。解法——在 LVGL 锁内、首次游戏绘制前：

```c
lv_display_enable_invalidation(display, false);      // 关闭失效标记
lv_timer_pause(lv_display_get_refr_timer(display));  // 暂停刷新定时器
```

LVGL 主任务、游戏动画定时器、按键、休眠计时器照常运行。注意：**串口截图会主动重画当前页，捕获的是游戏像素，看不见 LVGL 后来写入的覆盖——这类问题截图证明不了**，要靠宿主门禁（verify_display_ownership 模拟面板输出边界）。

## 5. 编译期布局不变量

元素跨横带边界时两条带重画频率不同 → 撕裂的字。防法：

```c
#define SCREEN_ELEMENT_FITS_BAND(y, h) \
    ((h) > 0 && (y) >= 0 && (y) + (h) <= SCREEN_H && \
     (y) / SCREEN_BAND_H == ((y) + (h) - 1) / SCREEN_BAND_H)
#define SCREEN_ASSERT_WITHIN_BAND(name, y, h) \
    _Static_assert(SCREEN_ELEMENT_FITS_BAND((y), (h)), \
                   #name " crosses a screen band boundary")
```

- 每个元素坐标文件头 `#define` + 紧跟断言（PokeWalk 落了 14+ 处）。
- 有意跨带的（sprite）用 `SCREEN_ASSERT_ALLOW_CROSS_BAND`（高度 ≤2×带高且必须重画所有涉及带）。
- **起因是真实事故**：不变量只活在注释里没有任何机制检查，用户实测"那片区域有时先渲一半再出来"。可机械检查的不变量要写成断言。

## 6. 页面表与导航（nav.c/h 模式）

不用 LVGL 页面系统，自建**显式页面表**：

```c
static const page_t PAGES[PAGE_COUNT] = {
    [PAGE_IDLE]  = {"P1 待机",  play_idle_enter,  play_idle_exit,  play_idle_key},
    [PAGE_BATTLE]= {"P3 战斗",  play_battle_enter,play_battle_exit,play_battle_key},
    // ...
};
```

- 每页只有三个函数：`enter / exit / key(btn, ev)`。
- 生命周期契约：`nav_go(P)` = 当前页 `exit()` → 切 `s_cur` → 新页 `enter()`；**exit 必须先停自己的 lv_timer 再删屏**（反了定时器访问已删对象）。
- 两种导航：`nav_go(p)` 链式跳转（清返回栈）；`nav_open(p)/nav_back(fallback)` 菜单下钻（`s_return[6]` 有界返回栈防递归超深）。
- **页面间传参用 nav_ctx 单槽**（链路上同时只有一条在处理）：传 **uid 而不是队列下标**——后台还在往队列塞、满了淘汰首条会使下标左移（真实症状："打的是 #64，抓到的是 #23"）。

## 7. 字体与 sprite（自研格式）

**字库格式**（`FNT1`）：14B 头（magic+ver+size+per+count）+ 升序 u16 码点索引（可二分）+ 定长字形（16×16 1bpp = 32B/字）。解析强校验 magic 与长度自洽；**count/per 从头读不写死**。半宽处理：`cp < 0x80` 时 advance = size/2 且字形左移 size/4（墨迹居中，只切空白）。详见 `fonts-chinese.md`。

**sprite 2bpp**：每字节 4 像素高位在左，色号 0~3 查 4 色 RGB565 调色板，**3 号透明**。整数倍放大（双线性会糊掉硬边缘，而硬边缘正是像素风的一部分）。工具函数：缩略图中心采样降采样、`ink_bounds` 扫描可见轮廓 bbox 做对齐、调色板读取集中一处（三份拷贝一份错极难发现）。

## 8. Web 同源预览（本路线的杀手锏）

**同一份固件 C 代码直接在浏览器里跑**，浏览器只接收屏幕字节。不是 Emscripten、不是手写 JS 绑定：

```
浏览器 firmware.html（页面选择/A B C 键/时钟步进）
  → HTTP POST /api/firmware {action: key|tick|page|reset|...}
  → 本地 server.py（ThreadingHTTPServer，仅绑 127.0.0.1，校验 Origin）
  → 独立 native 进程（stdin/stdout 行协议）
  → 【真实固件代码】nav.c / play_*.c / render.c / screen.c（约 50 个生产 .c 原样编译）
  → LCD transport 替身输出 RGB565BE 字节
  → 浏览器 Canvas.putImageData
```

关键设计：
- 本机 `cc -std=gnu11 -O2 -DHOST_BUILD=1` 编译生产源；资产用汇编 `.incbin` 生成 `_binary_*_start/end` 符号（与固件 CMake EMBED_FILES 符号名一致）。
- **构建缓存按内容寻址**：全部源码+头文件+夹具+资产 SHA-256 前 20 位作版本号。
- 帧协议：`FRAME <page> <ms> <mismatch>\n` + 240×320×2 RGB565BE 二进制。
- **宿主夹具保真 DMA 语义（精髓）**：LCD 替身对 `esp_lcd_panel_draw_bitmap` 只**保留指针**，直到生产 screen.c 发出 DMA 屏障（那个 NOP 同步点）才 memcpy 进全屏缓冲——**逼着被测代码遵守与真机相同的同步协议**；未同步的传输直接 `exit(3)`。`screen_host.c` 总共 6 行：`#include "../../../firmware/main/screen.c"`。
- lv_timer 模拟器：自建 16 个定时器槽 + `advance(ms)` 按到期序精确回调（暂停/单步就靠它）。
- Wi-Fi/NVS/音频/墙钟全是 fixture 服务（可控故障注入如 `save_failure_after`）；全屏 150KB 缓冲只存在于开发机。
- 诚实边界：预览不能代替真机验收 Wi-Fi 扫描、NVS 掉电、真实刷新耗时/色彩。

## 9. 双端共享渲染配方（防像素级漂移）

布局/比例/颜色只在 C 里算一次：
- 共享 C 场景配方：输入页面只读状态快照，输出有序绘制指令（`rect(x,y,w,h,rgb565)` + 文本回调）。
- 固件侧求交横带；Web 侧构建期原生执行同一 C 导出矩形 payload，JS 只做裁剪+写 Uint16Array——**不再有第二份求解**（web `Math.round` 与固件整数除法曾产生像素级差异）。
- **预算桥**：Python 表生成 `render_layout_budget.h`（`--check` 检测不同步），消除"坐标在 Python 和 C 各写一遍"的漂移。
- 验证：125 组输入 × 960 万像素逐值一致；负向对照（故意 x+1）必须检出首差。

## 10. 帧率策略

没有 60fps。tick 周期按内容定：照料动画 80ms、战斗回合 600ms、技能阶段 ≤18×60ms。脏区以横带为单位重画。"大色块+低帧率恰是 DMG 时代原生表达"——这是宪法级决策，不是妥协。

## 11. 颜色系统

九色 RGB565 实值常量（来源标注清楚，如 GSC UI 调色板）：`C_BG/C_INK/C_MID/C_LIGHT/C_HP_TRACK/C_FOCUS/C_HP_GREEN/C_HP_YELLOW/C_HP_RED` + `RGB(r,g,b)` 宏。分段判据**按填充像素不按百分比**（`fw >= w*50/100`）——"条宽改了不用重推阈值"（同一规则三处三个值是真实踩过的坑）。

## 12. 串口观测通道（自助验证两半）

- **截图**：`@@SHOT 240 320 rgb565be 80` 头 + 每条带 base64（每行 76 字符防串口缓冲丢行）；`tools/device/screenshot.py` 收成 PNG。回调式不缓存整屏。
- **按键注入**：`walk.py --keys e,c,A,b,a,a --settle 26`（小写单击大写双击）；**--settle 必须给足**（战斗最坏 43.2s，验结算用 `--settle 46`）。
- 两条硬边界：截图抓不到动画中间态；截图证明"画出来了"不证明"用对数据源"（换 sprite 要两图对比+bbox 度量）。

## 13. 何时不用这条路线

- UI 以菜单/列表/滚动文字为主 → 路线 A（LVGL）开发更快。
- 已有完整 C 引擎要移植 → 路线 C（自研渲染反而多余）。
- 不想维护字体/sprite 渲染器 → 路线 A 的 LVGL 字体生态省事。
