# 音频方案

> 硬件：ES8311 codec + I2S0 全双工（引脚见 `hardware.md` §5）。两个案例给出两条完整验证的路线：**运行时合成（零 PCM 存储）** vs **采样播放（ADPCM 压缩 + 单任务混音）**。

## 0. 硬件层要点（两路线通用）

- I2S 通道：TX-only，`dma_desc_num=6, dma_frame_num=240`，**`auto_clear_after_cb=true`**（DMA 空闲补零防啸叫）。
- 采样率 16000 或 22050 Hz，mono 16bit。
- codec 走 `esp_codec_dev`（DAC-only 模式，ES8311 作 slave，mclk_div=256）；音量用硬件 `esp_codec_dev_set_out_vol(0..100)`。
- sdkconfig 只开 `CONFIG_CODEC_ES8311_SUPPORT=y`。
- **开机炸音防护**：`bsp_audio_boot_quiet()` 放 app_main 第一件事。
- **独立播放任务是必须的**：codec 写入最多阻塞 1 秒，DMA 只有 65ms 深度——在 LVGL 任务或按键回调里播必掉帧。任务优先级**高于界面**（5 > 4）。

## 1. 方案一：运行时合成（PokeWalk，零 PCM 存储）

适用：chiptune 风格 BGM + 大量短音效 + 存储紧张。

- **一个混音器结构混三路**：`music_player_t`（4 声部 chiptune：note/noise/phase/increment/duty）+ `cry_player_t`（3 声部 GB 风振荡器，复刻金银叫声：envelope/duty/lfsr）+ SFX 效果器（含一阶 DC 滤波）。
- 音符数据从轨道数据编译成头文件（音乐数据 164B/首起步）——**音频零 PCM，Flash 几乎不占**。
- 参数（实测值）：块 480 样本、队列深 8、任务栈 4096、优先级 5、22050Hz。
- **队列纪律**：满则丢弃新请求不卡调用方；任务循环**排空队列只留最新**（"最新音效优先于过期的菜单点击，队列不能积压"）。
- **懒开关 codec**：静音/熄屏时 suspend，有事件才 init；`xQueuePeek` 阻塞等事件且不丢"检查与阻塞之间送达"的事件。
- 静音持久化 + 音量 10 档；`CONFIG_XXX_SILENT_BOOT` 静默构建（所有接口编译成空）。

## 2. 方案二：采样播放（Spire，IMA ADPCM 音乐 + PCM 音效）

适用：想要真实音乐/音色，Flash 有 ~1MB 余量。

### 存储（Flash EMBED_FILES）

- **音乐 = IMA ADPCM（4:1 压缩）**：60s 16kHz 音乐 ≈ 480KB。3 首共 ~1MB。
- **音效 = 裸 PCM16**（短促、随机访问）：每条 2~16KB，26 条共 ~250KB。
- `audio_meta.h` 由转换脚本生成（写死每条样本数，解码回环用）。

### 播放（单音频任务软件混音，10ms 块）

```
每次生成 160 样本（10ms）：
  每样本：
    audio_ima_next() 流式解码音乐（一个 nibble 一步，状态仅 7 字节）
    × music_gain（0..3200 淡入淡出，200ms）× duck_gain（音效闪避）
    + 当前音效 PCM 样本 → audio_mix_saturated() 饱和相加
    → esp_codec_dev_write()
混音公式：mixed = music*g1*g2 / (3*3200*1000)   （3 是预留削顶余量）
```

- **场景切换淡出淡入**：切歌只写 volatile 场景变量；音频任务发现不一致则每样本 gain-- 到 0 再换曲重新淡入——**切歌无爆音**。
- **音效闪避**：有音效时音乐增益每样本 -25‰ 降到 55%（降 45%），无音效回升。
- **空闲休眠**：无音效且音乐关时 `xQueueReceive(wake_queue, portMAX_DELAY)` 挂起，0 CPU。

### 音效调度器（纯 C 可测）

- 容量 8 的环形队列（portMUX 临界区）。
- **优先级**：胜负/破盾=3，奖励/购买=2，其余=1，导航=0。
- **合并**：连续 MOVE 只留队列里一个。
- **挤占**：队列满时新音效优先级 ≥2 可踢掉队首导航音；否则丢弃+LOGW。
- 单音效通道（一次一条，先后顺序天然保持）。
- `_Static_assert(AUDIO_EFFECT_COUNT-1 == GAME_SFX_COUNT)`——**枚举与素材表数量错位编译期拦截**（真机触发过的坑）。
- 上层语义：**动作音 + 结果音**两字段（先"出牌"声后"命中"声）——解决"一次更新只能带一个音效"。

## 3. 素材转换（Python 侧）

- 全部归一 16kHz mono i16（均值降混 + 线性重采样）。
- **响度归一**：目标 RMS=0.20、削顶 0.92——解决素材忽大忽小（真实坑）。
- 音乐裁剪按素材 loop 点选起点。
- **IMA ADPCM 编码器用纯 Python 重写**，与固件解码器严格互逆（89 项步长表 + 16 项索引表与 C 侧完全一致）。
- 循环：解码到 `sample_count` 尾部自然归零状态回绕 = 无缝循环。

## 4. 音频验收方法论（单独记，因为踩过大坑）

> S13 曾判 PASS（欠载 0/写错误 0/样本数吻合）而**用户听不到任何声音**——`set_volume` 全项目零调用，codec 停在硬件默认。"那些指标量的是『样本写出去了』，不是『有声音出来』"。

**修法 = 补能证明可听的手段**：
- **物理麦克风回采**（ES8311 全双工 `bsp_audio_read`）：静音 RMS 51 → 播放 RMS 3991（78×），峰值未触顶 clip 0。**下次判音频看这个，不看欠载计数。**
- Web 侧 WebAudio 现场合成，浏览器里能听"本该是什么声音"。
- 感知响度测试：关键音效 RMS ≥ 0.14、时长上限（ctest 门禁）。

## 5. 音频 checklist

- [ ] 播放在独立任务，优先级高于 UI
- [ ] codec 空闲 suspend（懒开关）+ DMA auto_clear
- [ ] 开机不炸音（boot_quiet 第一件事）
- [ ] 音量走硬件 vol，设置持久化，下限 >0（防"黑屏以为坏了"类比）
- [ ] 切歌淡入淡出无爆音
- [ ] 音效优先级 + 合并 + 挤占策略
- [ ] 枚举-素材表 `_Static_assert`
- [ ] 响度归一（RMS 目标值）+ 麦克风回采验收
- [ ] 静音构建开关（发布变体）
