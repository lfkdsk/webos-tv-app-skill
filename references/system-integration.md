# 系统集成

## 权限

| API | 开发者模式结果 |
|---|---|
| `com.webos.service.ime/insertText` 等 | `Denied method call` [说法] |
| `com.webos.service.config/getConfigs` 读 `tv.model.*` | `Access denied`(从 Moonfin 页面调用)[实测] |
| `com.palm.db/find`(自己的 kind) | 可用 [实测] |
| 应用列表、前台应用(网页应用里) | 不可用,要在设备外生成列表 [说法] |
| LS2 权限文件 | 需要 root,放 `/var/luna-service2-dev/*`,可能要重启 [说法] |

SSH 用户 `prisoner`,端口 `9922`。服务可以在方法里校验调用方:`m.ls2Message.applicationID().split(' ')[0]`。

## SSAP

- 用本机局域网 IP 的 3001 端口;回环地址报 invalid origin。[说法]
- 公共的已签名 manifest 被列入黑名单(`403 Pairing rejected`),用未签名的。[说法]
- `getForegroundAppInfo` 需要 `READ_RUNNING_APPS`。两处说法不一致,先试再定。
- 每次新配对都会弹确认框,出现重复弹窗时查日志。

## 文字输入

IME 被拒时,经 `http://127.0.0.1:9998/json` 用 CDP 的 `Input.insertText` 注入。系统应用和商店应用不暴露,所以只对自己的应用有效。[代码]

## 服务与开机

- 持有 activity 才能不被回收:`service.activityManager.create(...)`。
- 复活:`com.palm.activitymanager/create`,`{foreground:true,persist:true}`,`interval:'15m'`,完成时 `{restart:true}`。最长断 15 分钟;周期调度跨重启没有验证过。[说法]
- 开机拉起前先持久化标记(`boot_id` + btime),否则崩溃会循环重启。
- 判断其它服务是否在线:`com.webos.service.bus/signal/registerServerStatus`。未知状态不要显示成已停止。
- 目录要提前设好属主(`prisoner:jailer`)。

## 前后台

- 设了 `handlesRelaunch:true` 之后,系统不会自动把窗口提到前台:监听 `webOSRelaunch`,读 `e.detail`,调 `PalmSystem.activate()`。[实测,`ddf12cc`]
- 关闭应用回到 LG Home,不是上一个应用。要回去:先拉起目标,回调后 300ms 再关,2 秒兜底。
- 启动器从待机快速恢复时可能重复拉起,导致方向键失效;已在前台就跳过。
- `document.hidden` 时停掉视频和订阅。

## 启动器

- 自动列出所有 launch point,用 `hiddenApps` 隐藏、`dockApps` 置顶。列表要在安装新应用后重新同步。
- 图标优先 `largeIcon`,`custom-icons/<id>.png` 优先级最高。
- 网格用绝对定位的 `.item` 放进带 transform 的舞台,焦点 `scale(1.1)`。

## 屏保

订阅 `com.webos.service.tvpower/power/registerScreenSaverRequest`,回复 `responseScreenSaverRequest {clientName,ack:false,timestamp}` 可以阻止 LG 屏保。它约 2 分钟触发,盖在所有应用上。后台时先拉起屏保应用,按拉起结果决定 ack。[代码]

## 蓝牙(未在设备上测过)

`com.webos.service.bluetooth2/`:`a2dp/connect`、`a2dp/disconnect`、`adapter/pair`、`adapter/awaitPairingRequests`、`device/getStatus`。

- `adapter/startDiscovery` 约 10 秒自停,每 9 秒重启。
- 有些协议栈不发 `endPairing`,同时轮询设备列表,45 秒超时。
- 同一时间只有一个 A2DP 音频输出,先断开当前的。

## 其它

- Node 的信任库可能缺根证书(如 Apple 的),把 `ca.pem` 加入 `ca`,不要关校验。注意 `.gitignore` 的 `*.pem` 可能把它漏掉。
- 电视有线和 Wi-Fi 同时连接时可能出现 `ERR_NETWORK_CHANGED` 和卡顿。[推测]
- 不要依赖 IP:用 `<hostname>.local`,并为 Android 提供 IP 二维码。
- `os.networkInterfaces()` 在服务里被认为不可用,但另一个原生应用依赖它,两处冲突,先测试。
