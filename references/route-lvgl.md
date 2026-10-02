# 路线 A：LVGL 组件路线（esp_lvgl_port + LVGL 9.5）

> 蓝本：《尖塔远征》（`D:\aipassport-game\Spire-Expedition-main`），ESP-IDF 5.5.4 + LVGL 9.5.0 + esp_lvgl_port 2.9.0 + esp_codec_dev 1.6.2。适用：卡牌/文字冒险/菜单重/UI 密集型游戏。深度解剖见 `case-study-spire.md`。

## 1. 依赖声明（`main/idf_component.yml`）

```yaml
dependencies:
  idf: ">=5.5.4,<5.6.0"
  lvgl/lvgl: "9.5.0"
  espressif/esp_lvgl_port: "2.9.0"
  espressif/esp_codec_dev: "1.6.2"
```

`dependencies.lock` 提交仓库（锁组件精确版本 + hash）；`managed_components/` 不进仓库。

## 2. 移植三板斧（不手写 flush_cb / indev / tick）

```c
// ① 初始化 LVGL 任务（内部创建独立任务跑 lv_timer_handler，并提供互斥锁）
lvgl_port_cfg_t lvgl_config = ESP_LVGL_PORT_INIT_CONFIG();
lvgl_config.task_stack = 8192;               // 4K 会栈溢出（真机实证）
lvgl_port_init(&lvgl_config);

// ② 挂显示（单缓冲 20 行 DMA —— 无 PSRAM 下的现实选择）
const lvgl_port_display_cfg_t display_config = {
    .io_handle = s_panel_io,
    .panel_handle = s_panel,
    .buffer_size = 240 * 20,                 // 4800 像素 = 9.6KB DMA 内存
    .double_buffer = false,                  // ★ 单缓冲（双整屏 150KB×2 不可能）
    .hres = 240, .vres = 320,
    .color_format = LV_COLOR_FORMAT_RGB565,
    .flags = { .buff_dma = true, .swap_bytes = true },   // ★ SPI 大端字节序
};
s_display = lvgl_port_add_disp(&display_config);

// ③ 跨线程互斥
lvgl_port_lock(timeout_ms);   // 非 LVGL 任务碰 lv_* 必持锁
lvgl_port_unlock();
```

## 3. 三键 UI：不用 lv_group/lv_indev，自建页面状态机

**关键决策**：LVGL 的焦点模型为触摸/编码器设计，与"三键 + N 页 + 每页自管理选中索引"不匹配。正确架构：

- 状态结构体里每页一个 `selected_*` 字段（selected_card / selected_shop_item / ...）。
- `game_state_handle_input(state, event)` 按 `state->page` switch 分发到每页处理器。
- **上下键环形滚动**：`selected = (selected + 1) % item_count`；手牌 + "结束回合"虚拟槽（`item_count = hand_count + 1`，空手也能结束回合）。
- 选中态由渲染层表达（金色 2px 边框 vs 普通 1px）。
- 状态机返回**结构化 update**，渲染层照单执行：

```c
typedef struct {
    bool needs_render;
    game_render_scope_t scope;      // 三档渲染范围（见 §5）
    game_sfx_t audio_lead, audio;   // 先播动作音再播结果音（一次更新两个音效）
    game_state_message_t message;   // 枚举消息，渲染层 switch 出 const 字符串
    bool progress_changed, settings_changed;
} game_state_update_t;
```

## 4. 分层（可 PC 测试的根本）

```
纯逻辑层（0 依赖 ESP-IDF/LVGL，PC 可编译测试）:
  game_core      战斗规则原子操作
  game_content   只读内容表 + 路线生成/校验
  game_save      blob 编解码（纯函数）
  game_state     页面状态机（依赖 device_input.h 枚举——host 测试用 #ifdef HOST_TEST typedef 桩）
硬件层: device_input/display/i2c/battery/audio
装配层: game_app（唯一同时 import LVGL + 设备 + 状态机的组件）+ app_main
```

跨平台头手法：`device_input.h` 在 host 测试时 `typedef int esp_err_t; typedef void* QueueHandle_t;`，设备编译时才 include 真头。

## 5. 渲染性能：三档渲染范围 + 持久控件

**三档范围**（状态机决定，渲染层不反推数据变化）：

| 档 | 时机 | 做法 |
|---|---|---|
| `GAME_RENDER_FULL` | 页面切换 | `lv_obj_clean(screen)` 清空重建 + 重置所有持久控件缓存指针 |
| `GAME_RENDER_BATTLE_DYNAMIC` | 数值变化 | 血量/意图/状态原位更新（不重建） |
| `GAME_RENDER_BATTLE_HAND` | 切牌 | 只刷手牌层（3 个卡槽文本/边框/透明度） |

- **战斗图层只创建一次**：`if (s_battle_hand_layer == NULL || !lv_obj_is_valid(...))` 创建，之后原位更新。频繁 `lv_obj` 创建销毁是切牌卡顿的根因（CONTEXT.md 实证）。
- **数值飘字用固定对象池**：3 个循环复用的 label，杜绝频繁创建销毁。
- 主循环对非 FULL 刷新计时，**>50ms 打 WARN**（性能回归量化监控）。

## 6. 布局常量表（数据驱动 UI，Figma 对齐）

每个控件的 x/y/w/h 都是编译期 `const` 结构体，与设计稿一一对应：

```c
const game_ui_v3_battle_layout_t GAME_UI_V3_BATTLE = {
    .enemy_status = {8, 8, 120, 78},      .enemy_sprite = {140, 44, 92, 80},
    .player_status = {116, 104, 116, 80}, .player_sprite = {12, 104, 88, 80},
    .cards = { {28,210,54,60}, {92,206,56,64}, {158,210,54,60} },  // 中张放大突出
    .dialogue = {12, 196, 216, 74},       .end_turn = {72, 272, 96, 20},
};
```

颜色集中在 10 个常量（黑金像素风）：bg `#0b090e` / 面板 `#1a1015` / 高亮金 `#f4bc33` / 边框 `#8e6125` / 红 `#d94b3f` / 绿 `#4cc06a` / 正文 `#f5ddb0`。

**白闪教训**：所有页面从 (0,0) 画**不透明全屏暗背景**（半透明/负坐标裁剪容器会导致白闪）；加巡检断言"背景必须 (0,0) 铺满且不透明"。

## 7. LVGL 动效封装（全部可抄）

| 函数 | 效果 | 参数 |
|---|---|---|
| `animate_hit` | 受击抖动 | x±5px、55ms、3 次 |
| `animate_lunge` | 前冲 | 140ms 出 + 180ms 回 ease_out |
| `animate_pulse` | 缩放脉冲 | scale 256→288（LVGL9 的 256=100%）、150/220ms |
| `animate_target_hit` | 抖动+闪色 | image_recolor OPA_COVER 渐隐 TRANSP 360ms |
| `animate_selected_card` | 选中上浮 | y-6px、110ms |
| `animate_map_node` | 呼吸 | OPA 70↔100、420ms 无限循环 |
| `animate_floating_value` | 飘字 | 上移 18px+淡出，对象池 |
| `animate_battle_end_transition` | 胜败横幅 | 800ms 上移 + 350ms 淡出 |

纪律：动画前 `lv_anim_delete(obj, cb)` 防叠加；`lv_obj_is_valid()` 防野指针；飘字显示时 `lv_obj_move_to_index(label, -1)` 提到最前（被面板遮挡的真实坑）。

## 8. sdkconfig 关键项（LVGL 由 Kconfig 全权接管）

```ini
CONFIG_IDF_TARGET="esp32c3"
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y
CONFIG_ESPTOOLPY_FLASHFREQ_80M=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y       # UART0 TX=GPIO21 与背光冲突！
CONFIG_COMPILER_OPTIMIZATION_SIZE=y         # -Os
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_160=y
CONFIG_ESP_MAIN_TASK_STACK_SIZE=3584
# LVGL
CONFIG_LV_COLOR_DEPTH_16=y                 # RGB565
CONFIG_LV_MEM_SIZE_KILOBYTES=64            # LVGL 内部堆（40~60 个控件 24K 不够）
CONFIG_LV_USE_BUILTIN_MALLOC=y
CONFIG_LV_DEF_REFR_PERIOD=33               # ~30fps（官方 20=50fps 上限）
CONFIG_LV_USE_DRAW_SW=y                    # 纯软渲染（C3 无 2D 加速）
CONFIG_LV_DRAW_SW_SUPPORT_RGB565A8=y       # 素材格式必须开
CONFIG_LV_USE_FONT_COMPRESSED=y
CONFIG_LV_TXT_ENC_UTF8=y
CONFIG_LV_FONT_MONTSERRAT_14=y             # 默认字体兜底
# 只开用到的 widget（LABEL/BUTTON/IMAGE/BAR/LINE），EXAMPLES/DEMOS 全关
# esp_codec_dev 只开 ES8311
CONFIG_CODEC_ES8311_SUPPORT=y
```

## 9. 线程安全约定

- 任何非 LVGL 线程调 `lv_*` 必持 `device_display_lock(timeout)`。
- 主循环 `if (device_display_lock(0)) { render...; unlock; }` —— **锁不到就跳过本帧，绝不阻塞游戏逻辑**。
- 输入/音频对界面只做非阻塞入队（`xQueueSend(..., 0)`）。

## 10. 主循环骨架（game_app 任务）

```c
while (true) {
    TickType_t wait = pdMS_TO_TICKS(presentation_active ? 40 : 1000);  // 动画期 40ms 轮询
    BaseType_t got = xQueueReceive(s_input_queue, &event, wait);
    if (presentation_active()) { tick_presentation();         // ① 演出队列逐段推进
        if (got) continue;                                     // 演出期间锁输入
        ... }
    if (!got) { poll_battery_30s(); continue; }                // ② 空闲电池轮询
    if (s_battery_alert) { dismiss_alert(); continue; }        // ③ 低电弹窗优先吃键
    if (persistence_pending()) { persist_step(); continue; }   // ④ 结算持久化状态机
    game_state_update_t u = game_state_handle_input(&s_state, event, now);
    update_audio_scene(); play(u.audio_lead); play(u.audio);   // ⑤ 场景音乐+双音效
    if (u.progress_changed) persist_progress_then_clear_run(); // ⑥ 先进度后清 run
    if (u.settings_changed) save_settings();                   // ⑦ 设置/音量/背光
    if (u.needs_render) save_run_after_update();               // ⑧ 页面级自动存档
    if (u.needs_render && device_display_lock(0)) {            // ⑨ 三档渲染
        render_by_scope(u.scope); device_display_unlock(); }
}
```

## 11. 战斗演出队列（"播动画时锁按键"的标准解法）

出牌/敌方行动产生**固定长度分段播放队列**（6 项 × 64 字符文本）：

```c
#define BATTLE_PRESENTATION_MAX_ITEMS 6
#define BATTLE_PRESENTATION_TEXT_CAPACITY 64
#define BATTLE_PRESENTATION_ACTION_MS 520    // "使用了【打击】" 520ms
#define BATTLE_PRESENTATION_RESULT_MS 720    // "熔岩史莱姆受到9点伤害" 720ms
```

`start()` 一次状态更新构建队列（动作项→结果项→每个附加效果一项）；`tick(elapsed_ms)` 推进索引返回是否有变化；**播放期间主循环丢弃输入**。此模块纯 C，PC 可测。

## 12. 何时不用这条路线

- 逐像素精灵动画/复古像素风优先级最高 → 路线 B（横带软渲染更省、更可控）。
- LVGL 64KB 内部堆 + 8KB 任务栈是固定开销；极简游戏（Doom 移植）连 LVGL 都不要 → 路线 C。
- 需要官方仓库生态（Wi-Fi 配网/低功耗框架/BLE）→ 直接 fork 官方仓库开发（其 BSP 即 LVGL 路线，另有 `bsp_lvgl_*` API 与 demo 分支参考）。
