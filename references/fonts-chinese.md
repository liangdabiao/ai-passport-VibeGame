# 中文字体方案（240×320 小屏 + 8MB Flash）

> 核心事实：全量中文字库（GB2312 一级 14px 2bpp）≈ 数百 KB Flash，不可接受；且游戏文案会持续新增。**字体必须是构建产物（扫描源码生成子集），缺字必须是构建期错误而不是屏幕上的空白框。** 两个案例 + 官方仓库给出了三套已验证的管线。

## 0. 为什么这是头号难题

- UTF-8 正确、编译成功、字符串存在，**都不代表能显示中文**（默认 Montserrat 无中文字形）。
- 真实事故：新文案"二阶段"不在字体子集里 → 真机显示方块（Spire）；"鉴"字因字体集合选错子字体全空白 + 27% 字形是港台写法（PokeWalk 用了 PingFang.ttc 默认 index=0=香港繁体）。
- 缺字必须可见（LVGL `CONFIG_LV_USE_FONT_PLACEHOLDER=y` 开发期开着）+ 可检测（门禁）。

## 1. 方案选择

| 方案 | 路线 | 工具 | 代表 |
|---|---|---|---|
| LVGL 压缩字体子集（2bpp） | A（LVGL） | `lv_font_conv`（npm，锁版本） | Spire：14px 正文 + 24px 标题 |
| 自定义 1bpp 位图字库（FNT1 格式） | B（自研渲染） | Python Pillow 生成 .bin | PokeWalk：16×16，727 字形 42.4KB |
| 官方字体管线（16/24/32 三档） | 官方仓库 | `tools/*/gen_font.py` + lv_font_conv | 三字经游戏 |

## 2. 字符清单从哪里来（关键：单一来源，不手维护）

**扫描一切会上屏的来源**：
1. 源码字符串字面量（正则 `[\u3400-\u9fff]`，**跳过注释**——中文注释不花 flash）
2. 游戏文案/名字/台词的文本文件（如 `sanzijing.txt`、`strings.py`）
3. 数据资产里的名字（从入库产物 bin 读，不从中间文件——tmp 清理曾让字库从 420 字掉 297）
4. 手写标点集：`、。，《》【】！（），：；？·—…""''↑↓▶◀%+-/×`

**宁可多收不少收**：多收一字 32 字节，少收一字是真机空白+重烧。

## 3. 管线 A：lv_font_conv（LVGL 路线，Spire 版 49 行脚本）

```bash
# 1. 扫描源码收集字符 + 标点集
# 2. 字体源用 LVGL 组件自带的（不用额外下载）：
#    managed_components/lvgl__lvgl/scripts/built_in_font/SourceHanSansSC-Normal.otf（OFL 1.1）
npx lv_font_conv --size 14 --bpp 2 --format lvgl \
  --font SourceHanSansSC-Normal.otf \
  --range 0x20-0x7E --symbols "<扫描到的全部字符>" \
  --lv-font-name game_ui_font_zh_14
# 3. 生成后自动把 include 替换为 "lvgl.h"，统一换行符
```

- **2bpp**（4 级灰度抗锯齿）+ `CONFIG_LV_USE_FONT_COMPRESSED=y`：比 1bpp 好看、比 4bpp 省 50%。
- 标题字体单独生成：24px、只含标题几个字、不含 ASCII。
- 结果参考：14px 全集 ~700 字 + ASCII ≈ 50KB。
- 生成命令和字符清单进版本管理；每个字号分别生成不同名字（`app_font_16/20`）。

## 4. 管线 B：自定 FNT1 位图字库（自研渲染路线）

```
头部 14B: magic "FNT1" | ver u16 | size u16 | per u16 | count u32
码点索引: count × u16 升序（可二分查找）
字形数据: count × per（16×16 1bpp = 32B/字，每行 2 字节高位在左）
```

- 16×16 而非 12×12 的论证：实测"妙"12px 着墨 20% 糊成一团、16px 着墨 23% 左"女"右"少"清晰可辨；209 字体积差 2.8KB——**清晰度值这 2.8KB**。
- 解析纪律：**每个 REC_SIZE 从文件头读，不写死**；强校验 magic + 长度自洽（`14 + cnt*2 + cnt*per == len`），不自洽整表置空。
- 半宽 ASCII：定长 16×16 无 advance 字段 → `cp < 0x80` 时 advance = size/2 且字形左移 size/4（墨迹居中只切空白）。

## 5. TTC 集合字体陷阱（PokeWork 真实事故）

- PingFang.ttc 有 12 个子字体，Pillow 默认 index 0 是**香港繁体**。
- 后果："鉴"全空白（HK 面没这字形）+ 142 字形港台写法——**字数/文件大小/缺字差集统计全正常**，极难发现。
- 修法：`FONT_CANDIDATES` 传 `(路径, index=2, "PingFang SC Regular")` + 子字体自检字 `SANITY_CHARS = "鉴龙"`（"鉴"验证简体存在、"龙"验证不是繁体"龍"）。

## 6. 缺字门禁（必须有，两个案例殊途同归）

- **Spire**（`test_rendering_contract.py`）：提取源码全部字符串字面量非 ASCII 字符，与字体文件注释里的 symbol 集合做差集，缺任何一个测试失败。
- **PokeWalk**：`fw.sh build` **每次构建自动重生成字库**（代价 2 秒 PIL）——"加一句中文日志就可能引入新字"这类失败被构建吸收；另有 `render_selftest()` 启动时验证码点索引升序 + 7 个探针字符（每个都是历史上真缺过的："鉴""A""["）+ 2 个排版锚点（`"[A]照料"`=56px 等与 Python 表逐值一致）。
- **官方**：gen_font.py 对每个码点**先在源字体验证存在**再转换，缺字形直接报错。

## 7. 字体不存在的字符（▸ ♥ ✦）

不进字库（收进去只会得到隐形符号），改用算法生成点阵进 ui 素材（`pixelart.py`）。

## 8. LVGL 侧集成检查单

```ini
CONFIG_LV_TXT_ENC_UTF8=y            # 且显式关闭 ASCII 编码器
CONFIG_LV_USE_FONT_PLACEHOLDER=y    # 开发期让缺字可见
CONFIG_LV_USE_FONT_COMPRESSED=y     # （用压缩格式时）
CONFIG_LV_FONT_FMT_TXT_LARGE=y      # 32px+ 字体位图索引超 u16 时必须开
```

- 控件必须**显式** `lv_obj_set_style_text_font(label, &app_font_20, ...)`；设置主题/默认字体不会替换已有控件显式指定的字体。
- 检查"当前实际用的字体"：`lv_obj_get_style_text_font(label, LV_PART_MAIN)` + `lv_font_get_glyph_dsc()` 查码点（非 placeholder）。
- 混排 fallback 方向要明确（先查应用字体，缺字才 Montserrat）；两边都没有的还是显示不了；同字号不同字体基线/行高不同，混排要查行高。
- `lv_label_set_text()` 复制字符串；`set_text_static()` 要求缓冲生命周期覆盖使用期（栈局部缓冲不行）。
- UTF-8 不按任意字节截断（结尾 NUL 空间 + 字符边界）。

## 9. 排版预算（超宽=截字=事故）

- 提示行/名字/数值都定**像素预算**（如名字槽 120px），Python 表与 C 头文件之间用生成脚本桥接（`gen_layout_budget.py --check` 检测不同步）。
- 按模型上限算不按样本算：只查 2 字招名会漏 5 字"尖刺加农炮"256px 溢出。
- 巡检断言（LVGL 路线）：任何可见 label 高度 ≥ 字体行高（防截字）；任意两 label 矩形不重叠。

## 10. 快速决策树

```
用 LVGL？ ── 是 → lv_font_conv 2bpp 子集（§3）+ 缺字差集测试（§6）
        └─ 否（自研渲染）→ FNT1 1bpp 定长（§4）+ 构建自动重生成（§6）
字号？正文 14~16px；标题单独 24px+；16×16 1bpp 中文可辨识下限
来源字体？思源黑体（OFL）或 LVGL 自带 SourceHanSansSC；TTC 必须指定子字体 index + 自检字
```
