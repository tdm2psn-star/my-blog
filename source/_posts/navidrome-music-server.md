---
title: 用Navidrome搭建自己的音乐网站
date: 2025-12-19 10:00:00
updated: 2026-10-27
permalink: posts/navidrome-music-server/
tags:
  - Navidrome
  - Cloudflare Tunnel
  - systemd
  - FFmpeg
  - nftables
categories:
  - 运维实践
cover: /img/4.jpg
description: 用一台老笔记本搭建个人音乐服务器，记录 Navidrome 安装、局域网与公网访问、FLAC 离线转换、资料库清理及备份维护的完整实践。
---

家里有一台 Intel Core i3 M350 的老笔记本，还有一批从旧硬盘备份下来的音乐。我想做的事情很简单：把这些音乐重新整理起来，在家用手机和电脑收听，出门时也能通过自己的域名访问。

于是，我重新部署了 Navidrome。但网站能打开之后，真正的问题才陆续出现：在家访问自己的域名，为什么还会绕公网？FLAC 能在列表里显示，为什么播放不正常？转成 MP3、移走原文件后，为什么列表里又多了一批灰色歌曲？

最后采用的方案没有继续增加组件，而是把几个环节拆开：**在家走局域网，外出走隧道；FLAC 留作原件，MP3 用于日常播放；音乐文件、应用数据库和部署配置分别管理。**

下面从访问链路、安装配置和使用维护三个方面，介绍这套个人音乐服务的搭建过程。

<!-- more -->

## 一、整体方案：一台主机，两条访问链路

Navidrome 负责扫描本地音乐、管理用户和歌单，并向浏览器或兼容客户端提供音频。它运行在家里的笔记本上，Cloudflare Tunnel 则为它提供外部访问入口。

我的博客虽然已经使用腾讯云 EdgeOne Pages 托管，但音乐服务没有随之迁移。静态博客和需要登录、查询资料库的音乐服务，是两条不同的部署链路。

以下环境与配置以 2026 年 10 月的部署为准：

| 项目 | 当前情况 |
| --- | --- |
| 操作系统 | Ubuntu 26.04.1 LTS，x86_64 |
| CPU | Intel Core i3 M350，2 核 4 线程 |
| 内存 | 系统可见约 7.6 GiB |
| 安装方式 | 官方 Debian 软件包 |
| Navidrome | 0.64.2 |
| FFmpeg | 8.0.1 |
| 服务程序 | `/usr/bin/navidrome` |
| 配置文件 | `/etc/navidrome/navidrome.toml` |
| 应用数据 | `/var/lib/navidrome` |
| 播放音乐目录 | `/home/sky/old_hhd_bak/music` |
| FLAC 原件保存目录 | `/home/sky/old_hhd_bak/flac` |
| 监听设置 | `0.0.0.0:4533` |
| 局域网入口 | `http://192.168.0.19:4533` |
| 外部访问入口 | `https://music.cbsbm.cn`，经 Cloudflare Tunnel |

整理后的播放目录有 91 个 MP3、12 个 M4A 文件；66 个 FLAC 原件单独归档，不参与日常播放库扫描。

### 两条访问链路

```text
在家收听：
手机 / 电脑
    → 家庭局域网
    → 192.168.0.19:4533
    → Navidrome
    → 播放目录中的 MP3 / M4A

外部访问：
浏览器 / 支持的客户端
    → https://music.cbsbm.cn
    → Cloudflare 边缘入口
    → Cloudflare Tunnel
    → 本机 cloudflared
    → http://localhost:4533
    → Navidrome
```

**在家也可以用公网域名，但那不等于局域网直连。** 没有额外设置局域网分流解析时，使用 `https://music.cbsbm.cn`，仍按公网入口和隧道这条路径访问。需要明确避开这条链路时，直接填写局域网地址。

局域网地址是 HTTP，仅适合自己信任的家庭网络，不应在不可信网络中传输登录信息。更换路由器或主机 IP 后，需要更新客户端地址；固定使用时可在路由器上为笔记本配置 DHCP 地址保留。曲库应仅保存自己有权使用的音乐，配置公网入口也不意味着获得了公开传播或提供下载的授权。

## 二、安装：先恢复数据，再考虑是否需要编译

之前的文章主要介绍源码编译。重新部署时，我改用了官方软件包：对于这次恢复个人音乐库的需求，减少构建依赖比保留一套定制编译环境更实用。

这里也需要区分三个容易混淆的环节：

| 环节 | 主要关注什么 |
| --- | --- |
| 扫描与入库 | 能否读取文件、提取标题/歌手/专辑等信息 |
| 服务端输出 | 发送原文件，还是通过 FFmpeg 转码 |
| 客户端播放 | 当前浏览器或 App 是否支持收到的媒体格式 |

**标签库解决的是“这首歌叫什么”，不能单独解决“这个客户端能不能播放”。** 没有明确的源码修改需求，就不必把编译 Go、前端和标签库当成部署的必经步骤。

安装前先查看 [Navidrome 官方 Linux 安装说明](https://www.navidrome.org/docs/installation/linux/)和 [Releases](https://github.com/navidrome/navidrome/releases)，选择与处理器架构匹配的软件包，并核对发行方提供的校验值。下面使用本次部署的 `0.64.2` 举例，以后重装应重新确认版本和配置兼容性。

下载并验证对应 `.deb` 后，安装命令类似：

```bash
sudo apt install ffmpeg ./navidrome_0.64.2_linux_amd64.deb

navidrome --version
ffmpeg -version | head -n 1
```

如果是恢复旧系统，安装之前还要先寻找旧的 `navidrome.db`、配置和数据库备份。**音乐文件还在，不等于用户、收藏和歌单也还在。** 找到旧资料后，先留一份副本，再进行迁移。

## 三、配置：音乐与应用数据分开存放

我的配置文件位于 `/etc/navidrome/navidrome.toml`：

```toml
MusicFolder = "/home/sky/old_hhd_bak/music"
DataFolder = "/var/lib/navidrome"
Address = "0.0.0.0"
Port = 4533
LogLevel = "info"
DefaultLanguage = "zh-Hans"
EnableInsightsCollector = false
EnableLogRedacting = true
TranscodingCacheSize = "1GiB"

[Scanner]
Schedule = "0 * * * *"
WatcherWait = "30s"
ScanOnStartup = true
PurgeMissing = "always"

[Transcoding]
MaxConcurrent = 2
MaxConcurrentPerUser = 1
EnableCancellation = true

[Backup]
Path = "/var/lib/navidrome/backups"
Schedule = "0 4 * * *"
Count = 7
```

这份配置主要做了几件事：

- 音乐文件与数据库分开，替换音源不等于重建用户资料。
- 监听全部 IPv4 网卡，兼顾本机隧道回源和局域网直接访问。
- 扫描计划为每小时一次，并配置了启动扫描和文件监视等待时间。
- 转码全局上限为 2、每用户上限为 1，限制转码进程对老 CPU 的占用。
- 每天凌晨 4 点计划备份数据库，保留 7 份。机器需要在计划时刻保持运行，并定期检查备份文件。

`Scanner.PurgeMissing = "always"` 是这台机器为清理旧 FLAC 缺失记录作出的明确选择，并非所有音乐库都应该照抄，后面单独说明。

完整参数可对照 [Navidrome 官方配置选项](https://www.navidrome.org/docs/usage/configuration-options/)。更换版本时，也应检查这些选项是否仍适用。

## 四、systemd 管服务，音乐只给读权限

软件包安装的服务以 `navidrome` 专用用户运行，而不是使用日常登录用户或 root。当前启动入口是：

```text
/usr/bin/navidrome service execute -c /etc/navidrome/navidrome.toml
```

当前还有一个额外配置文件：

```text
/etc/systemd/system/navidrome.service.d/override.conf
```

内容为：

```ini
[Service]
SupplementaryGroups=sky
ProtectHome=read-only
ReadOnlyPaths=/home/sky/old_hhd_bak/music
```

它的目的，是在现有目录权限下给服务附加读取音乐所需的组权限，同时把 `/home` 对该服务设为只读。数据库、缓存等可写数据放到 `/var/lib/navidrome`。

这里的 `sky` 是本机用户组，照着部署时需要换成自己的权限方案。除了文件可读，还要保证服务能够逐级进入父目录：

```bash
namei -l /home/sky/old_hhd_bak/music

systemctl show navidrome \
  -p User -p SupplementaryGroups -p ProtectHome -p ReadOnlyPaths
```

如果需要新建 override，应使用 `sudo systemctl edit navidrome`，不要直接改软件包安装的主 unit。修改完成后：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now navidrome
sudo systemctl restart navidrome

systemctl is-active navidrome
systemctl is-enabled navidrome
```

不要通过 `chmod -R 777` 来绕过权限问题。播放器只需要读取音源，没有必要顺带获得修改整份音乐备份的权限。

## 五、为什么选 0.0.0.0，而不是只绑局域网 IP

如果只写：

```toml
Address = "192.168.0.19"
```

局域网可以访问，但这个监听并不覆盖 `127.0.0.1`，原先指向 `localhost:4533` 的 Tunnel 回源就需要另行调整。

现在使用 `0.0.0.0:4533`，让 Navidrome 同时接收本机和局域网连接，再用防火墙限制谁能连上来。`0.0.0.0` 是监听地址，不是要填进客户端的访问地址。

我为 4533 端口配置了单独的 nftables 规则：

```nft
table inet navidrome_filter {
    chain input {
        type filter hook input priority filter; policy accept;

        iifname "lo" tcp dport 4533 accept
        ip saddr 192.168.0.0/24 tcp dport 4533 accept
        tcp dport 4533 drop
    }
}
```

这组规则只针对 4533：允许回环接口，允许家庭 IPv4 网段，其余到该端口的 TCP 流量丢弃。它不是整台主机的完整防火墙，也不应直接覆盖系统中其他业务的规则。

规则文件放在 `/etc/nftables.d/navidrome-lan-firewall.nft`，通过独立的 systemd 服务加载，使其在重启后恢复：

```ini
# /etc/systemd/system/navidrome-lan-firewall.service
[Unit]
Description=Restrict Navidrome access to localhost and home LAN
Before=navidrome.service
After=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
ExecStartPre=-/usr/sbin/nft delete table inet navidrome_filter
ExecStart=/usr/sbin/nft -f /etc/nftables.d/navidrome-lan-firewall.nft
ExecStop=-/usr/sbin/nft delete table inet navidrome_filter
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

确认系统已安装 nftables，保存文件并检查语法，再启用服务：

```bash
sudo nft -c -f /etc/nftables.d/navidrome-lan-firewall.nft
sudo systemctl daemon-reload
sudo systemctl enable --now navidrome-lan-firewall
```

如果主机已经使用 UFW、firewalld 或统一管理的 nftables 配置，优先在现有规则体系中加入等价限制，不要不加判断地叠加另一套防火墙。

维护时还应检查内核中真正生效的规则：

```bash
sudo nft list table inet navidrome_filter
```

配置文件只是预期，运行中的规则才决定实际行为。启用后应同时从允许的局域网设备和其他网段测试；还要留意已有规则是否会影响连接。

此外，Cloudflare Tunnel 从本机连接 Navidrome，会经过回环允许规则。防火墙不会替代 Navidrome 自身的登录认证；服务在公网仍必须使用独立账户和强密码。

## 六、外部访问：复用 Cloudflare Tunnel

这台笔记本上已经运行着 cloudflared，因此不需要为音乐网站另外建立一套穿透服务。在 Cloudflare 控制台的现有 Tunnel 中，添加或确认 Published application 路由：

```text
music.cbsbm.cn → http://localhost:4533
```

其中主机名是 `music.cbsbm.cn`，服务类型选择 HTTP，目标为 `localhost:4533`。这里填写的是本机 Navidrome 的 HTTP 服务，不要因为外部使用 HTTPS，就把回源协议也误填成 HTTPS。

这个方案不要求给路由器增加音乐端口映射，但公网访问仍然依赖家庭主机、家庭网络和隧道。笔记本睡眠、断网或服务停止，外部访问就会中断。

配置完成后，先检查 Navidrome 与 cloudflared 是否运行，再测试域名入口。第一次初始化时，及时创建管理员并设置强密码；之后给日常使用者单独建立普通账户，不要多人共用管理员。

不过，隧道解决的是“外面能不能连进来”，不是“任何网络下都能流畅听歌”。我的实际使用中，局域网收听没有同样的卡顿，而经公网域名播放明显更慢。因此现在采用以下策略：

- **在家优先局域网直连。** 避开不必要的公网和隧道路径。
- **公网域名保留为外部入口。** 不把它当成已经完成性能优化的低延迟音乐服务。
- **不凭一次请求判断网络路径。** 节点位置和链路表现可能随网络与时间变化，排障时应看实际请求，而不是把一次测试结果永久套用。

Cloudflare 的账号凭据、Tunnel token、证书和用户密码，应与可以公开的配置示例分开保管，避免随脚本、仓库或截图泄露。

## 七、FLAC 与 MP3：把兼容成本从每次播放移到入库前

之前遇到过“FLAC 在列表中可见，但播放不正常”的问题。它至少有两类可能：

1. 当前浏览器、操作系统或播放设置与原始文件不兼容，或者服务端转码失败。
2. 客户端能够解码，但传输吞吐或稳定性不足，导致播放卡顿。

两者不能混为一谈，也不能简单归结为“浏览器都不能播 FLAC”。排障时，应分别测试原始格式与转码输出，结合播放请求、客户端能力和 FFmpeg 日志定位，而不是只看歌曲有没有入库。

对这台老机器而言，已经实施的工程取舍是：**保留 FLAC 原件，把日常播放副本提前转换成 MP3，原件移出扫描目录。**

2026 年 10 月 19 日留下的转换日志记录了：

```text
编码器：libmp3lame
参数：-q:a 2
新转换：39 个
跳过已有 MP3：27 个
失败：0 个
```

`-q:a 2` 是质量导向的 VBR 设置，不是固定 320 kbps。因此界面上显示 152、188、191 kbps 等不同平均码率，并不自动说明转换失败。

单个文件的转换示例：

```bash
ffmpeg -n -i "歌曲.flac" \
  -map 0:a:0 -map_metadata 0 \
  -c:a libmp3lame -q:a 2 \
  "歌曲.mp3"
```

这里使用 `-n` 防止覆盖已有文件；该示例只映射音频，不复制附带图片流，封面需要单独核对。转码后的 MP3 是有损副本，不应替代原始 FLAC 备份。原件要等输出能读取、标签与时长检查完毕，并实际试听之后再归档，不要一转完就删除源文件。

当前目录结构是：

```text
/home/sky/old_hhd_bak/
├── music/          # Navidrome 扫描：MP3 / M4A 日常播放副本
└── flac/           # FLAC 原件，不在当前 MusicFolder 内
```

预转换不会神奇地解决公网网络问题，但可以避免兼容客户端在每次播放时都要求服务器重新做同一种格式转换。也要注意：即使文件已经是 MP3，如果客户端选择了更低码率，服务端仍可能执行转码。

实时转码不一定每次都从零开始，缓存可能复用结果；但是否命中与客户端、输出设置及缓存状态有关。预转换的价值是把常用播放副本固定下来，用磁盘空间换取更稳定的播放路径，而不是声称实时转码必然很慢。

## 八、为什么 FLAC 移走后，列表里还留着灰色歌曲

文件消失和数据库条目删除是两件事。Navidrome 会保留缺失记录，以避免挂载盘暂时离线时直接丢失相关资料；管理员可能因此看到灰色曲目。官方的 [Missing Files 说明](https://www.navidrome.org/docs/usage/library/missing-files/)介绍了这一行为。

本机现在使用：

```toml
[Scanner]
PurgeMissing = "always"
```

这样在扫描时清理确认缺失的记录，而不是只标记 missing。**清理旧记录不等于把旧曲目的收藏、评分、播放次数和歌单引用自动转移给新 MP3。** 如果这些资料重要，应先备份、核对关联关系，再决定如何处理。

尤其是外接硬盘或网络盘，不建议不加判断地照抄 `always`。挂载失败时，缺失可能只是临时状态；这类环境可考虑保留缺失记录，或只在确认磁盘在线后的完整扫描中清理。

需要离线执行完整扫描时，可以使用下面的维护流程。**先确认音乐盘已挂载、目录内容正常，并理解缺失记录清理的影响，再执行。**

```bash
(
  set -e

  # 先生成数据库备份；失败则不继续。
  sudo -u navidrome /usr/bin/navidrome backup create \
    -c /etc/navidrome/navidrome.toml

  # 人工确认此处能看到预期音乐，不是空目录或掉盘后的挂载点。
  find /home/sky/old_hhd_bak/music -maxdepth 1 -type f | head
  read -r -p "确认音乐目录正常并继续扫描？输入 YES：" answer
  [ "$answer" = "YES" ] || exit 1

  sudo systemctl stop navidrome
  trap 'sudo systemctl start navidrome' EXIT

  sudo -u navidrome /usr/bin/navidrome scan -f \
    -c /etc/navidrome/navidrome.toml
)
```

停止服务是为了避免独立扫描器与正在运行的服务同时操作数据库；退出时尝试重新启动服务。维护期间会有短暂不可用，执行后还要用 `systemctl is-active navidrome` 检查启动是否成功。

完成后重新打开客户端，确认旧缺失条目是否清理、MP3 是否可播放；如果之前把 FLAC 加入过歌单，还需要检查歌单中的对应条目。

## 九、备份必须分成三份理解

### 1. 音源备份

包括 FLAC 原件、MP3/M4A 播放副本和必要的转换清单。原件与播放库放在同一块磁盘的不同目录，只是逻辑分离，**不是异机备份**。

### 2. Navidrome 数据库备份

数据库包含用户、播放记录等应用数据。当前已配置定时备份，也可以手动创建：

```bash
sudo -u navidrome /usr/bin/navidrome backup create \
  -c /etc/navidrome/navidrome.toml \
  -d /var/lib/navidrome/backups
```

根据 [Navidrome 官方备份说明](https://www.navidrome.org/docs/usage/admin/backup/)，内置备份只备份数据库，不包含音乐和配置。

定时配置不能替代备份检查。尤其是家庭笔记本可能在凌晨休眠或关机，应定期查看实际生成的文件：

```bash
sudo ls -lh /var/lib/navidrome/backups
```

还应把有效备份复制到另一块盘或另一台机器，并定期验证能否恢复。

### 3. 部署配置备份

至少包括：

```text
/etc/navidrome/navidrome.toml
/etc/systemd/system/navidrome.service.d/override.conf
/etc/nftables.d/navidrome-lan-firewall.nft
/etc/systemd/system/navidrome-lan-firewall.service
```

隧道凭据应在安全位置单独保管，不能混进准备公开的博客附件。恢复数据库前先停止服务，使用当前安装版本 `navidrome backup restore --help` 确认操作方法，并保留恢复前的现有数据，不要覆盖后才发现备份不适用。

## 十、排障顺序：先本机，再局域网，最后公网

```bash
# 1. 服务与开机启动
systemctl is-active navidrome cloudflared navidrome-lan-firewall
systemctl is-enabled navidrome cloudflared navidrome-lan-firewall

# 2. 监听端口
ss -lnt | grep ':4533'

# 3. 本机服务
curl --noproxy '*' --max-time 10 -i http://127.0.0.1:4533/ping

# 4. 局域网地址；最好再用另一台家庭设备验证
curl --noproxy '*' --max-time 10 -i http://192.168.0.19:4533/ping

# 5. 公网入口
curl --noproxy '*' --max-time 20 -i https://music.cbsbm.cn/ping

# 6. 应用日志
journalctl -u navidrome -n 100 --no-pager
```

`--noproxy '*'` 用于避免命令行代理干扰本次直连检查，不代表所有用户都应该永久关闭代理。

| 现象 | 优先检查 |
| --- | --- |
| 本机也无法访问 | 服务状态、配置解析、4533 监听、应用日志 |
| 本机正常，局域网失败 | 主机 IP、客户端是否同网段、防火墙、无线访客隔离 |
| 局域网正常，域名失败 | cloudflared 状态、DNS、Tunnel 回源地址和端口 |
| 页面能打开，某首歌失败 | 文件读取、媒体格式、客户端能力、播放请求与转码日志 |
| 局域网流畅，公网卡顿 | 音频实际下载吞吐、家庭上行、隧道链路和客户端缓存 |
| 删除文件后出现灰色歌曲 | 数据库缺失记录与 PurgeMissing 策略，不是浏览器刷新次数 |

最后的验收必须包含真实播放：至少完整听一首常用 MP3，再抽查 M4A、拖动进度、切歌及客户端重新登录。HTTP 200 和首页截图都不能替代这一步。

## 结语

这次部署最大的收获，不是又装好了一个服务，而是把“音乐文件还在”“网站能打开”和“日常听起来顺畅”分成了三个需要分别解决的问题。

官方软件包降低安装维护成本，systemd 约束服务权限，防火墙限制直接访问范围，Navidrome 管理资料库，Cloudflare Tunnel 则补上外部入口。每个组件只承担自己适合的职责。

对于这台老笔记本，我最后留下的原则很简单：**原件要留，数据库要备份；在家直连，外出再走隧道；能在入库前处理的兼容问题，不必留到每次播放时重复处理。**

它不需要成为一套大型流媒体平台。能把已有的音乐可靠地整理、保存和播放起来，就已经是一台旧电脑很合适的第二份工作。
