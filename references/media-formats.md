# 视频与音频格式

## 选管线

| 来源 | 管线 | 参数 |
|---|---|---|
| DASH | 服务生成 SegmentBase MPD,shaka 4.16 + MSE | `abr.enabled:false`,`bufferingGoal:40`,`rebufferingGoal:2`,`bufferBehind:20`,`maxAttempts:8`,`timeout:40000`,`stallTimeout:20000` |
| HLS | hls.js + MSE | 前向 60-120s,后向 30s,起始带宽估计 5 Mbps |
| 直播 fmp4 | shaka,`bufferingGoal:12` | |
| 直播 ts | 原生 `video.src`(支持情况为 [说法]) | |
| 单文件 | 原生 `video.src` | 先确认音轨 |

线路容灾放在服务里做时,关掉播放器的 ABR。

## 能力探测

- 逐轨用 `MediaSource.isTypeSupported` 并缓存。用过的字符串:`hev1.1.6.L120.90`、`hev1.2.4.L153.90`、`av01.0.08M.08`、`dvh1.08.07`、`ec-3`、`fLaC`。[代码]
- HDR 轨(qn 125/126)再看 `matchMedia('(dynamic-range: high)')`。[代码]
- **探测结果不等于能出声。** 对 C5,`canPlayType('video/mp4;codecs="hvc1.2.4.L153.B0,dtsc"')` 返回 `probably`。[实测]
- 加载失败时自动退回 AVC 重试一次。[代码]
- 杜比视界、杜比音效、Hi-Res 在海外哔哩里实现了探测,但没有真机确认。[说法]

## DTS

- LG 2025 款去掉了 DTS 解码和直通(媒体报道,C5 上行为一致)。[实测+报道]
- 症状:有画面、没声音、没错误,`video.audioTracks` 长度为 0,同片 AAC 版本正常。[实测]
- 不要照抄 EmbyTV 的 `AUDIO_CODECS = dts,dtshd,truehd`。[代码]
- Moonfin 上游在读不到 `tv.model.edidType` 时默认 MKV 里的 DTS 可播;C5 上从 Moonfin 页面读这个配置返回 Access denied,所以很可能被误判。这一步是推断。[实测+推断]

有转码权限时让服务器转码最简单,否则走软解。

## 软解音频(实测可行)

做法见 `Emby-Self/moonfin-patch/dts-softaudio/`:

1. 服务用自带的 ffmpeg 探测文件,默认音轨是 DTS 才启用。
2. 服务输出 `s16le`、48kHz、双声道 PCM,页面 `fetch` 流式读取。
3. 用 Web Audio 排程,视频继续走原生直放。

实测数据(C5):解码约 10 倍速,ffmpeg 约 5% CPU、7MB 内存,服务约 40MB;拖动后约 2 秒恢复声音。

### 音画同步要点

- **不要用 `<audio>`,** 它会让视频暂停。
- **`currentTime` 在 seek 后不可信:** 先报告目标位置,再退回更早的关键帧,实测差 1.3-2.4 秒。等 `playing` 之后再按实际位置对齐。
- 视频时钟和音频时钟有差异,约每 28 秒差 0.1 秒。超过 0.1 秒且连续 3 次再重新对齐,避免读数抖动造成爆音。
- 排程时减去 `baseLatency + outputLatency`。
- 已解码的数据多留一些:向后 8 秒、向前 20 秒,小幅回退不用重开解码。
- 用一个 `AudioContext`,重启时停掉节点,不要反复创建。
- 用 `AbortController` 中断旧请求。只取消 reader 时,连接会占满 Chromium 每个主机 6 个的上限,下一个 `fetch` 挂起且不报错。[实测]
- 倍速时没声音,这是简化取舍。

## 音轨切换(EmbyTV,未经真机确认)

顺序:`video.audioTracks` → `luna://com.webos.media/selectTrack {mediaId,type,index}` → 服务器。写完读回 `enabled` 确认。

- 索引是“同类型内的序号”,不是服务器的全局序号。
- `mediaId` 取 `video.mediaId`,否则查 `getActivePipelines` 的 `is_foreground`。
- 服务器禁用重封装时,换 `AudioStreamIndex` 返回的还是同一个文件。
- 国语和粤语语言码都是 `chi`,常常都标默认,`DisplayTitle` 也相同。要从 `Title` 判断方言。

## 字幕

- 文本字幕:侧车 `Stream.vtt` + `<track>`。跨域时 `<video>` 没设 `crossOrigin` 可能被拦,`file://` 下的行为未确认。[代码]
- 也可以自绘大字号字幕层,不依赖原生渲染。
- **ASS(SubtitlesOctopus):** libass 只用加载到的字体,默认兜底是拉丁字体,中文会变成方块。在包里放 CJK 字体并改 `fallbackFont`。不能直接读系统字体目录,XHR 返回 0;把 `/usr/share/fonts/DroidSansFallback.ttf` 拷进包里可用。[实测]
- ASS 首次加载可能要几十秒(服务器从 MKV 提取轨道),需要等待逻辑。[实测]
- PGS:EmbyTV 里配置和代码互相矛盾,没有定论。

## 播放器生命周期

- 释放解码器要彻底:销毁播放器、`removeAttribute('src')`、`load()`。
- `destroy` 之后的异步流程还可能继续:每个 `await` 之后检查 `destroyed`,否则脱离 DOM 的 `<video>` 还会出声。[实测,`df4cac4`]
- `document.hidden` 时暂停。
- hls.js 接管期间忽略 `video` 的 error,它有自己的回调。
- 错误逐级恢复:一次 `recoverMediaError` → 下一个源并保留位置 → 重试页面。
- 启动看门狗:每个候选 25 秒,超时换下一个清晰度。
- 同一时间只有一个 4K 解码器,做不了交叉淡化,用一个 `<video>` 渐黑切换。[说法]
