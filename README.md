# webos-tv-app-skill

LG webOS 电视应用手册,一份 Claude Code skill。从 6 个项目蒸馏:界面与遥控器、视频音频格式、页面与服务的数据流、打包调试、系统集成、修复方式。每条尽量标明是实测、代码实现还是只有说法。

主要设备是 LG OLED C5(webOS 25),开发者模式,不需要 root。换机型或版本后请重新确认。

## 安装

```bash
git clone git@github.com:lfkdsk/webos-tv-app-skill.git ~/.claude/skills/webos-tv-app
```

## 结构

```
SKILL.md                         核心认知、开发顺序、选型速查
references/
  ui-remote.md                   按键码、焦点、滚动、尺寸、性能
  media-formats.md               管线、探测、音轨、字幕、软解、音画同步
  data-service.md                页面与服务、令牌、存储、缓存
  packaging-debug.md             打包、传文件、CDP、补丁第三方应用
  system-integration.md          Luna 权限、保活、前后台、屏保、蓝牙
  bug-playbook.md                症状 → 原因 → 修法,已知未修的缺陷
```
