---
name: webos-tv-app
description: Playbook for building, patching and debugging LG webOS TV apps (web app + JS service, dev mode, no root), distilled from real projects on an LG C5. Covers remote/focus UI, video and audio formats, page-service data flow, packaging, CDP debugging and bug patterns. Use when writing or fixing a webOS TV app, patching an existing ipk, or debugging playback, focus, Luna, service or install problems on an LG TV. 中文触发:webOS、LG 电视应用、电视遥控器焦点、ares、ipk、Luna、电视播放没声音。
---

# webOS TV 应用手册

从 6 个项目蒸馏:海外哔哩(DASH + 选路)、一个私有 HLS 应用、航拍屏保、电视键盘、启动器、EmbyTV,以及给 Moonfin 打补丁的经验。主要设备是 LG OLED C5(webOS 25,SDK 11.2.0,Chromium 132,系统 Node 20)。

## 证据等级:照抄前先看

- **[实测]** 在电视上跑过,或有提交记录复现。
- **[代码]** 项目里这样实现了,但没有单独验证它在电视上的效果。
- **[说法]** 只来自注释、README 或提交说明。

这份手册记录的是这些项目的结论,不是 LG 的规格。换机型、换 webOS 版本后,[实测] 也要重新确认。

## 核心认知

1. **`<video>` 走的是电视自己的管线,不是 Chromium 的。** 桌面浏览器能放,不代表电视能放;反过来也一样。能力以真机为准。[说法+实测]
2. **`canPlayType` 会说谎。** C5 对 `hvc1...,dtsc` 回答 `probably`,实际没有 DTS 解码器,结果是有画面没声音、也不报错。[实测]
3. **同一时间只有一个媒体元素能放。** 第二个 `<audio>` 一开始播放,视频约 30ms 内就被暂停;Web Audio 不受影响。[实测]
4. **开发者模式能做的事比想象多。** 不用 root 也能跑 Node 服务、运行自带的静态 ARM 程序、用 CDP 调试。系统级 API(IME、部分配置读取)会被拒绝。[实测]
5. **页面只用 Luna 拉起服务,之后全走 127.0.0.1 HTTP。** 这是几个项目反复验证下来的结构。[实测]
6. **电视资源很紧。** 2GB 内存只剩约 300MB 可用,系统闲时负载也有 15 左右。别指望桌面级的并发和缓存。[实测]
7. **一切操作前先确认用户是否在看电视。** 安装、重启和测试都会打断观看。

## 开新应用的顺序

| 阶段 | 做什么 | 完成标准 |
|---|---|---|
| 骨架 | `appinfo.json`、esbuild IIFE、固定 1920×1080、按键路由 | 真机能启动,返回键能退到系统 |
| 焦点 | 虚拟 `.focused` + 几何导航 + 指针同步 | 遥控器和 Magic Remote 都走得通 |
| 服务 | Luna `ping` 返回端口和令牌,本地 HTTP | 服务被杀后页面能自动重连 |
| 播放 | 选管线,逐轨探测,错误逐级回退 | 真机放完一集,切走后没有残留声音 |
| 打包 | `ares-package app service --no-minify` | 重新安装后设置还在 |
| 调试 | CDP 脚本、深链启动参数、桌面 mock | 不碰遥控器也能复现问题 |

## 选型速查

| 场景 | 选择 | 详见 |
|---|---|---|
| DASH 点播 | 本地 MPD + shaka(MSE),关 ABR | media |
| HLS | hls.js(MSE),动态加载 | media |
| 单文件直放 | `video.src`,但要先确认音轨能解码 | media |
| 需要 Cookie、签名、Range、选路 | 放到 JS 服务里 | data |
| 电视不支持的音轨 | 服务解码成 PCM + Web Audio | media |
| 修改第三方 ipk | 解包后叠加脚本和服务,不动原 bundle | packaging |
| 把文件传到电视 | Node 临时接收器,不要用 `ares-novacom --put` | packaging |

## 参考文档

- [references/ui-remote.md](references/ui-remote.md):按键码、焦点、滚动、尺寸、性能
- [references/media-formats.md](references/media-formats.md):管线、探测、音轨、字幕、软解、音画同步
- [references/data-service.md](references/data-service.md):页面与服务、令牌、存储、缓存、API 代理
- [references/packaging-debug.md](references/packaging-debug.md):打包、安装、传文件、CDP、补丁第三方应用
- [references/system-integration.md](references/system-integration.md):Luna 权限、后台保活、前后台、屏保、蓝牙
- [references/bug-playbook.md](references/bug-playbook.md):症状 → 原因 → 修法,以及已知未修的缺陷

## 边界

- 不把令牌、Cookie、服务器地址写进仓库或日志;打印命令输出前先替换掉密钥。
- 第三方二进制(ffmpeg 等)不进仓库,记下版本和校验和,注意许可证。
- 不绕过内容的 DRM 或加密保护。
