# 症状 → 原因 → 修法

## 播放

| 症状 | 原因 | 修法 | 依据 |
|---|---|---|---|
| 有画面没声音,无错误 | 音轨是 DTS,电视解不了 | 换版本、转码,或软解 | 实测 C5 |
| 另开 `<audio>`,视频立刻暂停 | 只允许一个媒体元素播放 | 改用 Web Audio | 实测 |
| 拖动后声音提前 1-2 秒 | 视频退回更早关键帧,`currentTime` 先报目标值 | `playing` 后按实际位置对齐 | 实测 |
| 关掉播放器还有声音 | 销毁后异步流程继续,脱离 DOM 的 video 还在放 | 每个 await 后查 `destroyed`,彻底释放 | `df4cac4` |
| 第二次 `fetch` 一直挂起 | 旧流只取消 reader,占满 6 连接 | `AbortController`,或复用连接 | 实测 |
| `MEDIA_ELEMENT_ERROR: Format error` | 不一定是格式:曾是 URL 里多了一个大小写不同的参数 | 先核对 URL 和响应 | 实测 |
| 视频一直停在旧位置,新元素也一样 | 原因未确认(可能是残留的管线状态) | 让应用自己进入状态 | 实测未解 |
| 中文字幕变方块 | libass 兜底字体是拉丁字体 | 打包 CJK 字体 | 实测 |
| 字幕开头几十秒空白 | 服务器首次提取轨道 | 等待逻辑,放宽上限 | 实测 |
| 卡顿触发回退转码后失败 | 账号没有转码权限 | 健康检查不要自动触发回退 | 实测 |
| 快进卡顿 | 每次写 `currentTime` 都重开解码器 | 累积后提交一次 | 代码 |
| OSD 不消失 | 轮询提前返回,恢复后没重启计时 | `playing` 时重设 3 秒计时 | `b6b11f0` |
| OSD 停在 0:00 | effect 运行时 video 未挂载 | 依赖里加上 stream | EmbyTV |

## 系统

| 症状 | 原因 | 修法 | 依据 |
|---|---|---|---|
| 从启动器再次打开没反应 | `handlesRelaunch` 不会自动提到前台 | `PalmSystem.activate()` | `ddf12cc` |
| 服务调用超时 | 服务被回收,令牌变了 | 重新 ping 后重试 | 代码 |
| `spawn` 失败 | 没有执行权限 | `chmod` 或复制到临时目录 | 代码 |
| ffmpeg 报 `Failed to resolve hostname` | 静态编译不能用系统 DNS | 输入走本地转发 | 实测 |
| 证书错误 `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` | Node 缺根证书 | 配 `ca`,不要关校验 | `e7f7dbf` |
| 第二个来源所有请求 CORS 失败 | Emby 缓存串了 ACAO | 唯一 URL 参数 | `800118d` |
| 刚写入的状态查不到 | 服务器响应缓存 | 读取时加 `_` | 实测 |
| 卡片重叠 | 列宽算错 | 按 `总宽/列数` 算 | `65fbbab` |

## 排查顺序

1. **先确认真实状态:** 读 `currentTime`、`paused`、`videoWidth`、`error`、服务器会话的 `PlayMethod`。不要靠截图,视频平面截不到。
2. **再问用户屏幕和声音的实际情况。** 没声音时有没有画面,是判断方向的关键。
3. **分清是平台限制还是测试手法的问题。** 注入测试失败时,先怀疑自己的 URL 和代码。
4. **一次只改一个变量,** 每次测试都能自动恢复用户的播放。
5. **把验证过和没验证过的分开写。** 模拟环境通过不算电视上通过。

## 已知未修的缺陷(不要照抄)

读代码发现,没有在电视上复现。

**海外哔哩 webOS**
- 直播页没有 `df4cac4` 的修复:`live.js:219-232` 没有 `silence()` 和隐藏时暂停。
- 致命错误的自动重建实际不生效:`rebuildCount` 未初始化,`undefined++ < 2` 为 false。`player.js:286,316`

**EmbyTV**
- 播放器里的音轨菜单用方向键操作不了。`App.tsx:31`、`Player.tsx:455-460`
- 模态框没有真正困住焦点。`spatial.ts:202`
- `/System/Info/Public` 没加缓存参数。`client.ts:180`
- 启动时切换音轨失败不提示。`Player.tsx:113,213`
- 按 Home 不上报停止。没有 `visibilitychange`。
- 未加载的占位行不可聚焦,焦点可能卡住。`spatial.ts:210-213`
- `setScreenSaverEnabled` 没有调用方。`webos.ts:224-235`

**航拍屏保**
- `foreground` 方法在独立版服务里没有注册。`screensaver.js:228`
