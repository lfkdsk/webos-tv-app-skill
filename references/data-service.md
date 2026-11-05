# 数据与服务

## 结构

```
页面 ──Luna ping──▶ JS 服务(拿到 {port, token})
页面 ──HTTP 127.0.0.1 + token──▶ 服务 ──▶ 外部 API / CDN / ffmpeg
```

为什么不全走 Luna:`<img>`、`<video>`、流式响应都需要 URL。

## 规则

- **只监听 `127.0.0.1`,带随机令牌**(`randomBytes(16)`),防止电视上别的页面借用登录态。`EADDRINUSE` 时换端口。
- **保留 `PalmServiceBridge` 引用直到回调,** 否则可能被回收。[说法] 超时 20 秒。
- **启动不要被服务卡住:** 最多等 6 秒,然后继续显示界面。
- **自动重连:** 收到 403 或网络错误时重新 ping 并重试一次;回到前台时强制重新 ping,服务可能已经换了令牌。
- **POST 用 `text/plain`,** 跳过 CORS 预检。
- **服务要抗崩:** 捕获 `uncaughtException` 和 `unhandledRejection`。
- **保活和退出:** `activityManager.create('keepAlive')`;闲置 30 分钟且没有会话时退出。
- **`webos-service` 不打包,** 系统提供。只在非独立模式下 `require`,这样服务也能用普通 Node 跑测试。

## 存储

- 偏好和进度放 `localStorage`:存精简对象,捕获配额错误,批量写防抖,进度上限 300 条。读写都要容错。
- 服务数据放 `/media/internal/<appId>`,带可写探测和回退,写入用临时文件加 rename。[代码]
- 磁盘缓存:写 `.part` 再 rename,读时校验文件头,合并进行中的相同请求,未命中结果缓存 1 小时,按 mtime 淘汰(400MB 降到 300MB)。
- 下载要处理 `416` 和“Range 请求收到 200”;可用空间低于 1.5GB 时停止。`fs.statfsSync` 可能不存在,没有就跳过检查。
- 重新安装会清掉服务目录。[说法] 应用的 `localStorage` 和 DB8 在覆盖安装后保留:Moonfin 的 26 条 DB8 和 39 项 localStorage 都还在。[实测]
- 设备 ID 要稳定地存起来,否则每次启动都注册一台新设备。

## API 代理

- 服务里维护主机白名单和按接口的 Referer。
- Cookie 只存在服务里,页面拿不到;响应里剥掉敏感字段。
- 给第三方服务的请求不要带网页 Cookie。
- 跟随 302 时,认证头只发给原主机。媒体地址可能跳到存储服务器。[实测]
- 对同一主机限流和熔断,后台探测限 2 个并发,LIFO,跟随视口可见性。“未知”不要存成“不存在”。

## 媒体数据面

- 分块 2MB,预取视频 12 块、音频 4 块,淘汰最远的但保留第 0 块,最多 3 个会话,闲置 10 分钟清理。
- socket 上限:API 8 个,CDN 6 个。
- 静态编译的 ffmpeg 在电视上解析不了域名,让它从服务的 `/up` 转发取数据,转发要支持 Range 和跳转。[实测]

## Emby / Jellyfin

- **Emby 的响应缓存不按 Origin 区分,也忽略 `Cache-Control: no-cache`。** 多个来源(`file://` 和网页)共用服务器时,CORS 头会串,要加唯一的 `_` 参数并用 `cache:'no-store'`。图片 URL 带不可变 tag,可以缓存。[实测,Emby 4.9.5]
- 这个缓存也影响读状态:写入收藏后立刻查列表,看到的还是旧结果,要加 `_`。[实测]
- 直放 URL:`/Videos/{id}/stream?Static=true&mediaSourceId=...`,照抄播放器实际用的参数。曾因多加一个大小写不同的 `MediaSourceId` 报 `Format error`,去掉后正常。[实测]
- 账号策略可能禁用所有转码和重封装(看 `/Users/{id}` 的 `Policy`),这时只能直放。
- `api_key` 在 URL 里,打印日志前要替换掉。
