# airplay-go

用 Go 写一个把 **Linux(Kubuntu)系统声音稳定投到 Mac(AirPlay/RAOP)** 的工具,目标是不卡、可调延迟,达到/接近 AirMusic 的稳定度。

> 本文档严格区分四类信息:**① 已确证(抓包/命令实测)**、**② 推断假设(未验证)**、**③ 未知待逆向**、**④ 实现计划**。凡未亲自验证的,一律不当事实写。

---

## 背景一句话
系统自带的 PipeWire 也能投 AirPlay,但**网络抖动时会偶尔卡**(能自愈、不影响,但不够稳)。AirMusic(Android app)在同一台 Mac、同一 WiFi 下不卡。本项目想用 Go 自己实现一个更稳的发送端。当前 PipeWire 方案处于"有点卡但能自愈、可用"状态,作为过渡先用着。

---

## 一、已确证的事实(抓包 / 命令实测)

### 1.1 RTSP 握手(来自 `captures/am_ok_stream.txt`,AirMusic→Mac 成功连接抓包)
- 序列:`ANNOUNCE → SETUP → RECORD → SET_PARAMETER`(走 TCP 7000)
- **所有请求从 ANNOUNCE 起都带 `Session: 1`**(缺这个头,SETUP 返回 `520 Origin Error` —— Python 原型 caster.py 实测踩过)
- ANNOUNCE 的固定请求头(抓包所见):`CSeq`、`User-Agent: iTunes/11.3.1 (Macintosh; OS X 10.9.4) AppleWebKit/537.77.4`、`Client-Instance`(16 hex)、`DACP-ID`(16 hex)、`Active-Remote`(数字)、`Session: 1`、`Apple-Challenge`(base64 16 字节)、`Content-Type: application/sdp`
- ANNOUNCE 的 body 是 SDP,声明音频格式:
  ```
  m=audio 0 RTP/AVP 96
  a=rtpmap:96 AppleLossless
  a=fmtp:96 352 0 16 40 10 14 2 255 0 0 44100
  ```
- SETUP 请求头:`Transport: RTP/AVP/UDP;unicast;interleaved=0-1;mode=record;control_port=<本地>;timing_port=<本地>`
- SETUP 响应给出 Mac 的端口,例:`server_port=64391;control_port=50522;timing_port=0`
- RECORD 请求头:`Range: npt=0-`、`RTP-Info: seq=<起始seq>;rtptime=<起始ts>`;响应带 `Audio-Latency`
- **无密码时**:第一个 ANNOUNCE 直接返回 `200 OK`(响应里 `X-Apple-ProcessingTime: ~6000ms`,那几秒是 Mac 在等用户点 approve 弹框)
- **有密码时**:第一个 ANNOUNCE 返回 `401`,带 nonce,需用 HTTP Digest 重发:
  ```
  HA1 = md5("iTunes:airplay:" + password)
  HA2 = md5(method + ":" + uri)
  response = md5(HA1 + ":" + nonce + ":" + HA2)
  ```
  用户名固定 `iTunes`,realm 固定 `airplay`。

### 1.2 目标设备与环境(命令实测)
- Mac mDNS(`avahi-browse _raop._tcp`):`et=0,3,5`(**et=0 = 支持不加密**)、`cn=0,1,2,3`(支持 PCM/ALAC/AAC/AAC-ELD)、`vs=935.7.1`;主机名 `hhdeMacBook-Air.local`;IP `192.168.124.7`(DHCP,会变,靠 mDNS 发现)
- **AirMusic 实际用的**:ALAC 编码 + **不加密**(抓包里没有 `a=rsaaeskey`/`a=aesiv`)
- **PipeWire 默认用的**(`pw-dump`):`raop.audio.codec=PCM` + `raop.encryption.type=fp_sap25`
- 本机:IP `192.168.124.20`,无线网卡 `wlp0s20f3`
- WiFi 质量:ping 到 Mac 延迟在 `6~50ms` 之间抖动(0% 丢包,但抖动明显)
- 抓系统声音的命令:`parec --format=s16be --rate=44100 --channels=2 --device=@DEFAULT_SINK@.monitor`

### 1.3 卡顿相关的实测对比
- **AirMusic 在 250ms 延迟下不卡**(用户长期使用);**PipeWire 在同等条件下会偶尔卡** → 说明**差距在实现质量,不在缓冲大小**。
- PipeWire 卡死时的现场:`pactl` 显示 sink 状态 `RUNNING`,但 `ss` 显示到 Mac 的 TCP 连接数 = `0`,`tcpdump` 抓 15 秒 = `0` 个 UDP 包(它以为在播,实际啥都没发)。正常时约 `100 个 UDP 音频包/秒`。
- `pactl suspend-sink <raop> 1` 然后 `0`(挂起再唤醒)能踢掉僵死连接、恢复(实测有效)。
- 把 PipeWire 的 `raop.latency.ms` 从 250 加到 2000,**体感没改善**(没解决卡;原因未深究,故不写因果)。
- 把 PipeWire 强制改 `ALAC + 不加密` 后,自愈变差/更频繁卡;**回退默认 `PCM + fp_sap25` 后自愈恢复**。所以目前 PipeWire 的最稳配置 = 默认编码 + 250ms。

---

## 二、推断 / 假设(合理但**未验证**,写代码前要先验证)

- **[假设]** AirMusic 不卡,是因为它实现了 RAOP 的**丢包重传**(control 端口上,接收端请求、发送端补发);PipeWire 这块缺失或弱。
  → 依据:RAOP 协议层面有重传机制(公开已知);但"**PipeWire 到底有没有/做得多好**"未直接验证,这是推断。
- **[假设]** 卡的根因 = WiFi 抖动/丢包 + 重传缺失。WiFi 抖动是事实(1.2),"重传缺失导致卡"是推断。
- **[观察,原因待查]** PipeWire 改 `ALAC+不加密` 自愈变差、回退默认恢复(1.3)。原因未明(可能是 PipeWire 对 none/ALAC 路径实现较差,而非协议问题)。
  → 注意:这**不代表** Go 版不能走"不加密"——AirMusic 正是不加密且不卡,说明协议层不加密可行;PipeWire 改 none 变差是 PipeWire 的实现问题。

---

## 三、未知 / 待逆向(还没搞清,**不要当事实用**)

需要从 `~/Desktop/airmusic/captures/am_cap.pcap`(AirMusic 完整抓包,含音频/控制流)逐包逆出来:

- [ ] **RTP 音频包的确切字节布局**(头各字段、payload 起始、时间戳步进)— 待核对
- [ ] **重传请求 / 响应包的确切格式**(control 端口)— ⚠️ 最关键的缺口,Go 防卡靠它
- [ ] **sync 同步包 / timing 对时包的格式**(NTP 时间戳如何编排)
- [ ] ALAC 未压缩帧的封装细节(若用 ALAC)

> 说明:RTP 是标准协议(12 字节头:V/P/X/CC、M/PT、seq、timestamp、SSRC),RAOP 在此之上有自己的约定。标准结构可参考,但**具体字段取值/payload type/重传变体必须用本抓包核对**,不照抄记忆。

---

## 四、Go 实现计划

### 4.1 架构(goroutines)
```
main: mDNS 发现 Mac(_raop._tcp)→ RTSP 握手(net.Conn TCP)
 ├─ A 抓系统音频(exec parec/pw-record monitor → 编码)→ 发送队列
 ├─ B 按节奏打 RTP 包发 UDP 到 server_port,同时存入"已发送环形缓冲"
 ├─ C 监听 control 端口,收到重传请求 → 从环形缓冲补发        ← 防卡核心
 ├─ D 周期发 sync 包;响应 timing 端口对时
 └─ E RTSP keepalive + 连接健康检测,僵死/断开就自动重做握手
```

### 4.2 关键设计点
- **重传(C)** 是和 PipeWire 拉开差距的核心,必须做对(但格式还没逆出来,见 §三)。
- **不加密**:走 et=0(依据 AirMusic 抓包),跳过 FairPlay/fp_sap25,大幅降低实现难度。
- **连接自愈(E)**:检测到僵死(类似 §1.3 的"假 RUNNING")就自动重连,不像 PipeWire 卡死不动。
- **缓冲大小**:已知"小缓冲(250ms)+ AirMusic 不卡",所以**先把重传做对**,缓冲先用小值即可;是否需要更大缓冲属于后期调优,**没有实测依据前不预设结论**。

### 4.3 里程碑
- **M1 最小可用**:mDNS 发现 + 握手(不加密,照 §1.1)+ PCM 音频 RTP 发送 + 基础重传 → 跟 PipeWire 跑同一首歌对照,验证更稳
- **M2**:完善 sync/timing 同步 + 自动重连
- **M3**:ALAC 编码 + 延迟可调
- **M4**:UI(设备列表 / 连接 / 状态 / 延迟)

### 4.4 第一步(新对话从这开始)
1. 用 tshark 打开 `~/Desktop/airmusic/captures/am_cap.pcap`,**逆出 control 端口的重传请求/响应包格式**(填补 §三 最关键缺口)
2. 搭 Go 骨架,实现 M1
3. 对照 PipeWire 验证

### 4.5 可参考的 Go 库 / 实现
- mDNS 发现:`github.com/grandcat/zeroconf` 或 `hashicorp/mdns`
- UI(M4):Go 的 `fyne` / `wails`,或沿用 caster.py 的 Web UI 思路
- ALAC 编码:现成 Go 库少,大概率自己实现"未压缩 ALAC 帧"
- 参考实现(读其重传/同步逻辑):shairport-sync(C,接收端)、owntone 的 `raop_device.c`(C,发送端)、pyatv(Python,RAOP 客户端)

---

## 五、现有资产

仓库内:
- `captures/am_ok_stream.txt` — AirMusic 成功握手的完整 RTSP 文本

本地 `~/Desktop/airmusic/`(大文件,未入库):
- `captures/am_cap.pcap` — AirMusic 完整抓包(含音频/控制,逆 §三 用)
- `caster.py` — Python 原型:握手能通、认证过,但音频发送无重传 → 会卡(可参考握手代码 + Web UI)
- `raop_sender.py` / `raop_handshake.py` — 早期握手实验
- `base.apk` + `out2/` — AirMusic 反编译(核心 `out2/sources/app/airmusic/sinks/airplay/b.java`)
- `重连AirPlay.sh` — PipeWire 卡死时一键重连(`pactl suspend-sink <raop> 1; sleep; 0`),治标
- `AirPlay投声-使用说明.md` — 当前 PipeWire 过渡方案的使用说明

---

## 协议名词
- RAOP = AirPlay 1 音频协议;RTSP 信令(TCP 7000)+ RTP 音频(UDP)
- approve = Mac 弹的"是否允许连接",**在 Mac 上点**,通常首次/重连时出现
