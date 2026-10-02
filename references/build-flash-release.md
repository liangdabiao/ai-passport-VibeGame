# 构建 / 烧录 / 发布

> 覆盖：环境、sdkconfig 模板、分区表、日常构建、烧录安全、完整镜像、社区发布。

## 1. 环境

- **ESP-IDF 5.5.3**（官方基线；Spire 用 5.5.4 也验证可行）。本机安装：`D:\esp\esp-idf-v5.5.3`。
- 激活：`source <idf>/export.sh`（bash）或 `& <idf>\export.ps1`（PowerShell）；验证 `idf.py --version`。
- `idf.py set-target esp32c3`（fresh checkout / 换 target 后必须）。
- 加速：`idf.py --ccache build` 或 `export IDF_CCACHE_ENABLE=1`。
- `idf.py fullclean` 只清构建状态，不恢复 defaults 变更——defaults 改动后跑 `set-target` 重建。
- 依赖锁：`dependencies.lock` 提交仓库（组件管理器精确版本+hash）；改 `idf_component.yml` 后重新生成并一起提交。
- Python 侧资产管线另需：Pillow、fonttools、（音频处理）numpy。**构建脚本内嵌自动重生成字库**（2 秒 PIL）是值得抄的实践。

## 2. sdkconfig.defaults 模板（带注释的精选）

```ini
# --- 目标芯片 ---
CONFIG_IDF_TARGET="esp32c3"
# ESP32-C3 无 PSRAM，故不设 CONFIG_SPIRAM

# --- Flash 8MB ---
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y
CONFIG_ESPTOOLPY_FLASHFREQ_80M=y
# 固定镜像头，关闭 detect，便于 merge-bin
CONFIG_ESPTOOLPY_HEADER_FLASHSIZE_UPDATE=n

# --- 分区表 ---
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_FILENAME="partitions.csv"

# --- 控制台：USB-Serial-JTAG（C3 原生 USB, GPIO18/19）---
# 不用 UART0：其默认 TX=GPIO21 与本板背光引脚冲突！
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y
CONFIG_ESP_CONSOLE_SECONDARY_NONE=y

# --- 优化（游戏默认体积优先；引擎移植也可 -O2）---
CONFIG_COMPILER_OPTIMIZATION_SIZE=y
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_160=y
CONFIG_ESP_MAIN_TASK_STACK_SIZE=3584
CONFIG_FREERTOS_HZ=100

# --- 无线按需（游戏不用就关运行时）---
CONFIG_BT_ENABLED=n
# 注：官方仓库构建系统可能把 WIFI 重置回 y——接受"编译进固件但不 init"，运行时不占内存

# LVGL/字体/codec 等路线专属配置见 route-lvgl.md §8 与 fonts-chinese.md §8
```

检查生成的根目录 `sdkconfig`（不能只看 defaults）。

## 3. 分区表模板

**极简游戏固件（Spire，推荐默认）**：

```csv
# Name,   Type, SubType, Offset,   Size,     Flags
nvs,      data, nvs,     0x9000,   0x6000,
phy_init, data, phy,     0xf000,   0x1000,
factory,  app,  factory, 0x10000,  0x7F0000,
```

无 OTA / 无 Recovery / 无 otadata——独立游戏固件的极简选择。0x0~0x9000 留给 bootloader（分区表在 0x8000）。

**带大数据分区（移植路线，Doom）**：

```csv
nvs,      data, nvs,     0x9000,   0x6000,
phy_init, data, phy,     0xf000,   0x1000,
factory,  app,  factory, 0x10000,  0x300000,
wad,      data, 0x40,    0x310000, 0x4F0000,
```

**兼容官方小程序 BLE 安装的契约分区（PokeWalk，不能动）**：

```csv
nvs,      data, nvs,     0x9000,   0x6000,
phy_init, data, phy,     0xf000,   0x1000,
factory,  app,  factory, 0x10000,  0x340000,
cardid,   data, nvs,     0x356000, 0x4000,    # 上游契约
save_restore, data, 0x40, 0x360000, 0x10000,  # USB 导入暂存
wifi_config,  data, nvs, 0x370000, 0x6000,
recovery, app,  test,    0x700000, 0x100000,  # 上键 5s 进入，上游契约
```

**注意**：新增分区（如 save_restore）要求**同时更新 0x8000 分区表和 0x10000 应用**，不能只写应用。

## 4. 日常构建与增量烧录

```bash
idf.py build                          # 构建
idf.py -p COM15 flash                 # 增量烧录（只写应用，不清 NVS——进度保留）
idf.py -p COM15 monitor               # 监视
esptool.py -p COM15 write_flash 0x310000 data.bin   # 数据分区（首次一次）
```

**端口探测**：优先 pyserial 找 `vid==0x303A`（Espressif）；不要硬编码 COM 号/usbmodem 名（编号会变）。

**烧录安全（PokeWalk 的硬门禁，值得默认采用）**：
- **没有任何备份就拒绝烧**：设备出厂可能是别人的玩法，覆盖就没了。备份 = `esptool read_flash 0x0 0x800000`（整 8MB，几分钟）存 `.device-backup/flash-<时间戳>.bin`。
- 烧录前必须征得用户同意（设备连接不构成授权）。
- 数据线必须支持数据传输（不能是纯充电线）。

## 5. 完整 8MB 镜像（发布形态）

```bash
esptool.py --chip esp32c3 merge_bin --flash_mode dio --flash_freq 80m \
  --flash_size 8MB -o full_8MB.bin \
  0x0 bootloader/bootloader.bin 0x8000 partition_table/partition-table.bin 0x10000 app.bin
# 或 idf.py merge-bin（新版本）
```

- 产物 = bootloader(0x0) + 分区表(0x8000) + app(0x10000) + 空白数据区，**从 0x0 一次写入**。
- 烧录流程（PowerShell 范式，每步查 `$LASTEXITCODE`）：

```powershell
python -m esptool --chip esp32c3 --port COM15 erase_flash
if ($LASTEXITCODE -ne 0) { throw "擦除失败" }
python -m esptool --chip esp32c3 --port COM15 write_flash 0x0 ".\release\game_full_8MB.bin"
if ($LASTEXITCODE -ne 0) { throw "烧录失败" }
```

- **警告必须写进 README**：完整镜像烧录会清空 Flash、NVS 存档和设置；旧设备要走"保留存档的组件更新流程"或先 USB 备份。
- 校验链：`release/SHA256SUMS.txt` + README 内嵌主哈希；增量烧录时 bootloader/分区表/app 三段写入后回读哈希（CONTEXT.md 模式）。
- 验证固件布局：从 `flash_args` 读偏移，检查分区表 MD5、边界不重叠、app 不超分区（官方 `verify_firmware.py` 思路；`flash_args` 当数据，不当脚本执行）。

## 6. 交付与真机验收

- 每轮交付分开报告：`Build / Host tests / Device tests / Unverified` 四行。
- **编译通过 ≠ 硬件验证**；烧录成功 ≠ Device PASS——要按验收清单检查启动和功能。
- 真机回归清单（发布前）：启动日志（版本号/自检 PASS 数/任务启动）→ 三键全页走查 → 存档读写往返 → **玩家进度保持**（旧档升级后逐项核对）→ 静音/亮度设置保持。
- 发布证据存 `reports/evidence/<topic>/`（提交回执/哈希/烧录日志/启动摘要）。

## 7. 社区发布（ai-passport.folotoy.cn）

完整 checklist（PokeWalk 实战总结）：

1. **产物**：`package_firmware.py` 用 merge_bin 合成 full.bin（**不含真实 NVS/cardid/recovery 内容，空白填充**）+ manifest.json（SHA-256/大小/地址/校验状态）。
2. **官方校验**：跑官方校验器（检查组件一致性/分区 MD5/应用容量/恢复入口/保护区）。声明边界："结构通过官方检查 ≠ 已通过社区 BLE 安装/更新实测"。
3. **封面**：480×640（**3:4 竖版**）JPEG/PNG/WebP ≤10MiB。经验：用清晰的原始游戏帧（主体完整无遮挡），别用缩放糊图。
4. **文案**：双语标题与介绍，接口 **4000 字符上限**；版本限制如实写明，不夸大。
5. **发布**：官方 publisher 工作流（zip 下载）→ 社区登录"发布新玩法"→"用 AI 辅助发布"→ **授权码连接（授权入口默认折叠，无需交密码）**→ 先 `whoami`/`projects` 核对 → **已有草稿或待审版本则停止报告冲突** → 选固件/封面/源码地址/双语介绍 → 预览后上传。
6. **更新**：用官网"撤回并修改"编辑重提；**不要另建项目**。
7. **归档**：回执+哈希+验证存 `reports/evidence/`；归档到官方 `plays/` 是单独的双语文本 PR（不传固件）。

## 8. 无真机时的近似手段

- 官方浏览器模拟器（FoloToy-Passport-Simulator，ESP-EMU+WASM+QEMU）可跑 ESP32-C3 固件镜像——近似验证启动/渲染，不能代替真机外设（ADC 按键/音频/电量计）与时序。
- 宿主同源预览（见 route-native-c.md §8 / route-lvgl.md 的 UI 巡检）覆盖大部分 UI 迭代。

## 9. 发布节奏范本（Spire CONTEXT.md 模式）

每次发布 = 源码 + 文档同步 + host 测试 0 失败 + UI 巡检图 + `idf.py build` + 完整镜像生成与逐段校验 + 真机增量烧录（保 NVS）+ 启动日志确认 → 提交 Git → CONTEXT.md 记录 SHA-256 与验证证据。**CONTEXT.md 是倒序开发日志：改了什么+测试结果+产物大小+哈希+真机证据。**
