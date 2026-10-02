# 存档系统与状态压缩

> 存档是嵌入式游戏的生命线：掉电、版本升级、磨损、损坏存档带崩游戏，四个威胁都要有答案。两个案例给出了高度一致且互补的方案。

## 1. 为什么用 NVS 而不是自制双 buffer

自制 A/B 双槽+CRC+单调序号理由成立（任何时刻掉电至少一份完好）——**但 NVS 已提供同样保证**（每记录 CRC32、页级状态机、未完成写入挂载时丢弃）。自己做"先数据后 CRC"顺序、擦除块边界、磨损均衡，**三件事写错的表现都是"拔电偶尔丢档"——概率性、极难复现**。用被大量产品验证的实现。

- 用现成 `nvs` 分区（0x9000, 0x6000=24KB），不新增分区。
- 磨损计算翻转假设：存档 ~1-4KB 占 1 个 4096B 扇区，NOR 10 万次擦写——每遭遇存（28 次/天）能撑 **9.8 年**。**磨损不是约束，真正的约束是掉电丢多少进度。**

## 2. 布局：整块 blob + magic + 版本 + 校验和

**一个游戏状态 = 一个 blob**（避免 8 次写入 8 个失败点）：

```
[ magic 4B ][ version 1B ][ 数据 ... ][ checksum 4B (FNV-1a 32) ]
   "CGRN"       5                        hash=2166136261; hash^=b; hash*=16777619
```

- NVS 命名空间 + 少数几个 key：Spire `"cardgame"/{progress, run, settings}`；PokeWalk `"pokewalk"/{state 整块, opening 独立 u8}`。
- 独立小 key 的价值：开场"只播一次"标记用独立 u8，**页面无需写整块世界存档**（传 false 时绝不擦掉已写入的 true）。
- 读写纪律：每次 open→操作→close 不留长命句柄；**写三步 `nvs_set_blob` →（其它 key）→ `nvs_commit` 不能省**（set 只写缓存，返回成功≠落盘）。
- 读结果四态（PokeWalk）：`OK / EMPTY / ERROR / MIGRATED`——只有 EMPTY 允许新游戏写入；**ERROR 必须保留原 NVS 字节**（初始化失败不是擦玩家存档的许可）。
- `nvs_flash_init` 遇 `NO_FREE_PAGES/NEW_VERSION_FOUND` 直接报错，**不静默擦除**。

## 3. 版本演进：前缀保持式洋葱（最值得学的模式）

**加字段 = 新版本 struct = 旧 struct 打头**，迁移变成"有界前缀读"：

```c
typedef struct {
    union { save_v16_t v16; struct { /* v16 字段平铺 */ }; };
    rest_clock_t rest_clock;          // V17 新增
} save_t;
_Static_assert(offsetof(save_t, rest_clock) == sizeof(save_v16_t), "V17 preserves V16 prefix");
_Static_assert(sizeof(save_v5_t) == 2296, "legacy save layout changed");
// ... 每个版本一对"大小不变 + 前缀保持"断言 —— 存档版本 +1 时改了旧字段，编译不过
```

- 版本识别：blob 尾部版本字节 + blob 长度双重判定（Spire：V1=156 → V5=826，**版本升级只在尾部追加，老偏移永不改动**）。
- 逐级迁移：V5 补 inventory、V10→11 补 exploration、V13→14 **复用探索结构未用字节存 bitmask——存档大小不变的新字段优先复用预留字节**。
- V1 特殊路径：老格式没有路线数据 → `route_seed=1` 重建 + 落回安全页。

## 4. 解码后整体验证（防损坏存档带崩游戏）

校验每个字段范围（职业<3、卡数 1~40、堆总数≤40）、每张卡 id 在内容表中存在、路线合法（复用生成器/校验器分离的校验器）、页面状态一致性（REWARD 页必须 reward_count>0）。载入后容错：路线校验失败按种子重生成；旧格式归一到当前约束。

**防御性细节**：读入的 `_Bool` 字段先 `memcmp` 与 true/false 标准表示比对再取值（原始 blob 可能含非法 bool 表示，直接读是 UB）。

## 5. 事务性写入（产出与凭证消耗同临界区同落盘）

全部"获得奖励+消耗资源"合并进**同一个临界区 + 同一次落盘**：

- `world_capture_uid(uid, mon)`：收容、点亮图鉴、按 uid 出队在一个临界区完成，随后立即把三者一起存档。（真实缺陷：顺序曾是 receive→mark_caught(立刻写 NVS)→take_uid，中间掉电可重复捕获同一条。）
- `world_battle_loot_uid`："Once per won active session. Repeated calls return the same award without adding it again."
- `world_choose_starter`："Build and save a complete snapshot **before** publishing the leader. Failure leaves the pending choice unchanged; retry is safe."
- 通关持久化状态机（Spire）：**先存永久进度 → 成功后才清除当前冒险**，两步一个微型状态机，任一步失败停在结算页重试——避免"通关了但 run 还在"。
- 运行时会话不进存档（重启即失），但"已开战"的 pending 移除**先落盘**，防掉电后已打过的目标复活。
- **幂等键**：`encounter_t.exp_granted` + 锁内按 uid test-and-set（防"打完→逃跑→再打"无限刷经验）。

## 6. 自动存档分级

分级判据不是"重要"而是"**丢了能不能再拿回来**"：
- 捕获（尤其闪光）→ 立即存
- 经验/照料 → 5 分钟节流存
- 页面切换 → 整页存档（Spire：`needs_render` 即存，断电即续）

时间基准：`esp_timer_get_time()` 开机微秒（RTC 内部 RC ±5% 温漂）+ `last_uptime_us` 存"上次开机微秒"；跨关机需 Wi-Fi 校时 + UTC 锚点补偿。

## 7. 状态压缩（400KB SRAM 上装下完整 RPG）

| 技巧 | 实例 |
|---|---|
| 紧凑记录 | `mon_t` 12 字节：species/level/hp/intimacy u8 + explore u16 + nickname_idx/flags u8 + exp u32。**昵称存索引不存字符串**（预设名池） |
| 位图代替集合 | 图鉴 4×19B（seen/caught/shiny_seen/shiny_caught 各 ⌈151/8⌉）——"闪光遇到但跑了"的遗憾也要能存 |
| 定点代替浮点 | Q10（×1024）；C3 无 FPU。Q10 与浮点在 1061 次真实扫描判定零翻转 |
| 大数组 static | 扫描结果 64×76B、存档缓冲 3.7KB 全 static——3584B 主栈被 2.6KB 局部数组撑爆是真实 Guru Meditation |
| 布隆过滤器 | 8192 bit（1KB）记"见过的 AP"，与真集合零差异 |
| LRU 定长槽 | 地点记忆 8 槽 512B 代替数据库 |
| 现算不存 | 招式表按等级现算（含进化前祖先招）——"现算比存档省" |
| 存种子重建 | route_seed/rng_state 存档 → 路线/洗牌序列可复现（不用存整个生成结果） |

## 8. 存档工具（进阶，用户需要时做）

- **USB 备份协议**：独占行协议（如 `!PWBACKUP ...`）+ 会话令牌 + 心跳超时 + 分块 CRC + 宿主 ACK；畸形/超长行不下落到旧按键注入。
- **导入安全**：独立 `save_restore` 分区暂存新旧镜像，提交标记最后写入；重启时 NVS 初始化前安装并校验，失败回滚；无法回滚时**停止启动游戏**避免继续写坏存档。
- **网页端**：GitHub Pages + Web Serial（桌面 Chrome/Edge），存档只写用户选择的本地目录不上传。`.pksave` = JSON（设备标识/固件 hash/版本/CRC/base64 镜像）；**兼容边界是备份协议和分区布局，不是游戏版本号**。
- 新增分区（如 save_restore）要求**同时更新 0x8000 分区表和 0x10000 应用**，不能只写应用。

## 9. 存档设计 checklist

- [ ] blob 头有 magic + 版本 + 尾部校验和（FNV-1a 够用）
- [ ] 每版本 `_Static_assert` 锁大小与偏移
- [ ] 新字段只在尾部追加（或复用预留字节）
- [ ] 读侧四态：OK/EMPTY/ERROR/MIGRATED；ERROR 保留原字节
- [ ] 解码后逐字段验证 + 内容表存在性 + 范围
- [ ] 写侧：产出+消耗同临界区同 commit；幂等键防重放
- [ ] 自动存档按"丢了能否找回"分级
- [ ] 构建产物含游戏数据可重建（种子在档）
- [ ] 版本升级真机验证：旧档 → 新固件 → 进度保持逐项核对
