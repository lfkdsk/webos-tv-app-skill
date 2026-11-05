# 打包与调试

## 打包安装

```bash
ares-package dist/app dist/service -o dist --no-minify
ares-install -d <device> dist/<id>_<ver>_all.ipk
ares-launch -d <device> <id> -p '{"debug":"param"}'
ares-inspect -d <device> --app <id> --open
```

- 必须 `--no-minify`:已压缩的产物再压一次曾把输出搞坏。[说法]
- `services.json` 的 `id` 和服务名都是 `<appId>.service`,必须以应用 ID 开头。
- appinfo:`type:web`、`handlesRelaunch:true`、`disableBackHistoryAPI:true`。
- 可执行文件的权限位会被 `ares-package` 保留,服务里还是要防一手:先 `chmod`,跑不起来就复制到临时目录。[实测+代码]
- 安装后路径:`/media/developer/apps/usr/palm/{applications,services}/<id>`。
- 用户空间是 32 位 armhf(内核 aarch64),所以要用 armhf 静态程序。`/media/developer` 挂载为 `nosuid`,不是 `noexec`。[实测]
- macOS 的 BSD `ar r` 会写坏 ipk(只剩 96 字节),要改 ipk 就用 Python 重写。[说法]
- 开发者模式到期会清除侧载应用。[说法]

## 把文件传到电视

| 方式 | 结果 |
|---|---|
| `ares-push` / `ares-pull` | CLI 3.x 对电视不可用 |
| `ares-novacom --put` | 失败 |
| `ssh -p 9922 prisoner@tv` | 要 `+ssh-rsa`;批处理模式下认证被拒(原因没查) |
| **电视上跑 Node 临时 TCP 接收器,Mac 用 `nc` 发** | **可用,31MB 约 1 秒** |
| 电视连 Mac 的端口 | 被 Mac 防火墙挡住 |

小文件可以 `echo <base64> | base64 -d > f`。从电视取文件:用 Node 分片输出 base64,每片约 90KB,因为 `ares-novacom` 的 stdout 约 190KB 就会截断。传完校验 md5。[实测]

## CDP 调试

- `http://<tv>:9998/json/list` 列出可调试的页面,用 WebSocket 发 `Runtime.evaluate`。
- Node 需要 `--experimental-websocket`。
- 模拟按键:`Input.dispatchKeyEvent`,`rawKeyDown`/`keyUp`,`windowsVirtualKeyCode`。输入文字:`Input.insertText`。截图:`Page.captureScreenshot`,但**截不到视频平面**,视频状态要读 `currentTime`、`paused`、`videoWidth`。
- 服务日志:`ares-log -f`。启动器可以把按键码存进 `localStorage` 做环形缓冲。

### 脚本陷阱(都踩过)

- 大结果不要 `console.log` 后 `process.exit`,管道里的输出会被截断;写文件。
- `pkill -f name` 会杀掉 ares 启动的、命令行里带 `name` 的远程 shell;用 pidfile。
- zsh 不拆分 `$VAR`,多参数用数组或脚本放进 bash。
- `Runtime.enable` 会回放历史 console,统计时要区分新旧。
- 注入调试代码后,监听器会一直留着。测完 `location.reload()`,别留在用户的会话里。
- 第三方应用自己的状态会干扰注入测试(比如把视频拉回旧位置)。尽量让用户用它自己的界面进入目标状态,再只叠加要测的部分。

## 打补丁第三方 ipk

优先**叠加**,不改原 bundle:解包、在 `main.js` 前注入独立脚本、再带上服务一起打包。Moonfin 的 DTS 软解就是这样,升级时重打成本低。

不得不改 bundle 时:

1. 在压缩代码里做唯一的精确字符串替换,替换前断言只出现一次。
2. 对 `.js` 副本跑 `node --check`,记下补丁后的 md5。
3. 就地写应用目录会失败(开发者 uid 会变),要重新打包安装。
4. **安装前备份** DB8(`luna://com.palm.db/find`)和 localStorage,**安装后核对**数量和关键设置。
5. 记下怎么重做,因为应用更新会覆盖补丁。

## 测试

- 单元测试作为部署的门槛,把解析、签名、同步逻辑做成不依赖 DOM 的模块,用 Node mock 测。
- 桌面模式:mock `PalmServiceBridge`,服务 `--standalone`,配深链启动参数。
- 焦点和真实播放只能在电视上确认。给调试加上 `-p` 启动参数直达目标页面。
- 不要长时间盯着 UI 操作:用假的挂起地址(本地接受连接但不响应)可以把播放器卡在加载态,方便检查界面。
