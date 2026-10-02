# 资产管线（图片 / 数据 / 溯源）

> 原则：**资产是构建产物，不是手维护的文件**。上游真源 → Python 转换 → 自描述二进制/C 数组 → EMBED_FILES/rodata。两个案例的管线都证明：手画/手填的资产是事故之源。

## 1. 总体流向（PokeWalk 范式）

```
上游真源（不手填、不手画）
  例：pret/pokered 反汇编 gfx、PokeAPI 灰度变体、固定 commit 的数据仓库
        │ fetch_*.py（记录来源 commit/坐标/参数）
        ▼
assets/*_sources.json（溯源文件：每个像素/音符都能追到上游）
        │ convert_*.py（sprites/palettes/font/moves/ui/music/...）
        ▼
assets/*.bin（自描述二进制，EMBED_FILES 进固件，共 158.8KB）
        │ inventory_assets.py（资产断言，退出码当 CI 门禁）
        ▼
固件 assets.c 解析（magic + 长度自洽校验，逐字节读不强转指针）
```

**为什么溯源如此重要**：手绘大木博士被用户一眼否决（"和皮卡丘不是一个风格"），改用 pret 反汇编真品同格式同管线一次成功。数值同理：**不手填，从权威反汇编取**（曾凭记忆手填训练家数据错三处；中文名用官方译名——从英文回译八个全错）。

## 2. 二进制资产格式设计（小端、头部自描述）

| 项 | 设计 |
|---|---|
| 头部 | magic 4B + ver u16 + rec_size u16 + count u32（+ pool_size 如有名字池） |
| 索引 | 升序 u16 码点（可二分）或显式 id |
| 记录 | 定长；变长部分用偏移+长度指向尾部池 |
| 校验 | 解析时强校验 `头 + count×rsz + pool == 文件长`，不自洽整表置空 |

**铁律：每个 REC_SIZE 从文件头读，不写死**——convert 脚本注释曾把 32 写成 28，照抄会让第 2 条记录起错位 4 字节，而**错位后读出的仍是合法数值（不崩只是全错）**。

**RISC-V 注意**：逐字节 `rd16/rd32` 读，不强转指针——未对齐访问在 RISC-V 是异常不是静默慢速。

## 3. 图片转换（Spire 的 LVGL 路线）

- 两种输出：**背景 tile → `LV_COLOR_FORMAT_RGB565`**（不透明）；**角色/怪物/图标 → `RGB565A8`**（RGB565 颜色面 + A8 alpha 面，保留透明，比 ARGB8888 省 1/4）。
- **精灵先按 alpha 边界裁切再缩放**：`source.getchannel("A").getbbox()` 裁透明边 → `Image.Resampling.NEAREST`（**像素风必须最近邻**）等比缩放居中到目标框。
- 输出 C 文件：`static const uint8_t xxx_data[]` + `const lv_image_dsc_t xxx = {.header={.cf,.w,.h,.stride=w*2},...}`——直接被 LVGL image widget 引用。
- 体量参考：35 张图 → game_assets.c 39559 行 ≈ 550KB rodata。
- **许可白名单机制**：脚本内 ASSETS 表即"允许的素材映射"（源缺失直接报错）+ ASSET_CREDITS.md 声明许可（CC0/OFL）+ SHA-256 + 下载页。

## 4. 2bpp 精灵格式（PokeWalk 的自研路线）

- 每字节 4 像素高位在左，色号 0~3 查 4 色 RGB565 调色板，**3 号透明**。
- **同一编码多重语义显式处理**：2bpp 色号 3 在精灵外=透明、精灵内=白（高光）。不做区分的后果：人物图 idx3 占 78.7%，上屏整件白大褂消失。修法=洪泛填充判"内部"（**与画布边界不连通**才是内部，局部判断对凹形轮廓误判）。
- 调色板读取集中一处（三份拷贝一份算错极难发现）；shiny 变体偏移用 size_t 计算（防 u8 回绕）。
- 整数倍放大（非整数倍插值会糊掉点阵）。

## 5. 门禁断言输出性质不断言过程（三条铁律）

1. **同一条管线单一来源**：所有素材从同一上游走同一转换。
2. **同一编码多重语义显式处理**（见 §4 透明/白）。
3. **断言输出性质**：不写"调用了 flood_fill"，写"转换后不该透背景的区域像素≠背景色"——"守护绑在函数调用上，新脚本直接绕过"是真实复发。

## 6. 资产进固件

```cmake
# CMakeLists.txt —— 二进制嵌入（生成 _binary_<name>_start/end 符号）
idf_component_register(... EMBED_FILES "audio_generated/sfx_confirm.pcm" ...)
```

```c
extern const uint8_t _binary_sfx_confirm_pcm_start[] asm("_binary_sfx_confirm_pcm_start");
// 懒初始化成 {data, byte_count, sample_count, loop_sample, encoding} 数组
```

- 图片 C 数组直接编译；音频 bin 用 EMBED_FILES；都在 rodata，运行期指针直读零拷贝。
- 体量纪律：**资产总预算写进宪法**（PokeWalk 7 个 bin 共 158.8KB ≈ app 分区 5%）；超预算先裁剪/降色深，不 silently 挤占。

## 7. 文档也是资产（生成物）

- 页面说明文档从 `strings.py` 单一来源生成（`gen_pages.py`）——改文案要改单一来源再生成，不直接改文档。
- 布局预算头文件从 Python 表生成（`gen_layout_budget.py --check` 检测不同步）。

## 8. 素材许可合规

- 每个素材记录：作者、许可（CC0/OFL/Apache）、SHA-256、下载页（ASSET_CREDITS.md / AUDIO_CREDITS.md / *_sources.json）。
- 不将原作素材声称为原创或统一 MIT 授权；同人作品明确标注来源与边界（PokeWalk README 的"来源与验证边界"节是范本）。
- 字体：思源黑体/Source Han Sans（OFL 1.1）；下载锁 SHA-256。

## 9. 管线 checklist

- [ ] 上游真源固定 commit/参数，fetch 脚本可重跑
- [ ] 溯源 JSON：每个资产可追到上游坐标
- [ ] 二进制头部自描述 + 解析自洽校验 + REC_SIZE 不写死
- [ ] 像素风缩放只用最近邻/整数倍
- [ ] 透明语义（2bpp 3 号 / alpha 边界裁切）显式处理
- [ ] inventory 断言脚本退出码当门禁
- [ ] 许可清单 + SHA-256
- [ ] 资产总 KB 在宪法预算内
- [ ] 生成物（文档/预算头）有 --check 同步检测
