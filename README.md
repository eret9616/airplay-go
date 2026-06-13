# airplay-go

用 Go 写一个把 **Linux(Kubuntu)系统声音稳定投到 Mac(AirPlay/RAOP)** 的工具,目标是不卡、可调延迟,达到/接近 AirMusic 的稳定度。

> 本文档严格区分四类信息:**① 已确证(抓包/命令实测)**、**② 推断假设(未验证)**、**③ 未知待逆向**、**④ 实现计划**。凡未亲自验证的,一律不当事实写。

---

## 背景一句话
系统自带的 PipeWire 也能投 AirPlay,但**会偶尔卡**(网络抖动时)。它是一个"够用但不够稳"的方案。本项目想自己实现一个更稳的发送端。当前 PipeWire 方案处于"有点卡但能自愈、可用"的状态,作为过渡先用着。

---

## 一、已确证的事实(抓包 / 命令实测)

### 1.1 RTSP 握手(来自 `captures/am_ok_stream.txt`,AirMusic→Mac 成功连接抓包)
- 序列:`ANNOUNCE → SETUP → RECORD → SET_PARAMETER`(走 TCP 7000)
- **所有请求从 ANNOUNCE 起都带 `Session: 1`**(缺这个头,SETUP 会返回 `520 Origin Error` —— 早期 Python 原型 caster.py 实测踩过)
- ANNOUNCE 的 body 是 SDP,声明音频格式:
  ```
  m=audio 0 RTP/AVP 96
  a=rtpmap:96 AppleLossless
  a=fmtp:96 352 0 16 40 10 14 2 255 0 0 44100
  ```
- SETUP 请求头:`Transport: RTP/AVP/UDP;unicast;interleaved=0-1;mode=record;control_port=<本地>;timing_port=<本地>`
- SETUP 响应给出 Mac 的端口,例:`server_port=64391;control_port=50522;timing_port=0`
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

### 1.3 PipeWire 卡顿的故障现象(实测)
- 卡死时:`pactl` 显示 sink 状态 `RUNNING`,但 `ss` 显示到 Mac 的 TCP 连接数 = `0`,`tcpdump` 抓 15 秒 = `0` 个 UDP 包(即:它以为在播,实际啥都没发)
- 正常时:约 `100 个 UDP 音频包/秒`(tcpdump 计数)
- `pactl suspend-sink <raop> 1` 然后 `0`(挂起再唤醒)能踢掉僵死连接、恢复播放(实测有效)

---

## 二、推断 / 假设(合理但**未验证**,写代码前要先验证)

- **[假设]** AirMusic 不卡,是因为它实现了 RAOP 的**丢包重传**(control 端口上,丢包由接收端请求、发送端补发);PipeWire 这块缺失或很弱。
  → 依据:RAOP 协议层面有重传机制(公开已知);但**「PipeWire 到底有没有做重传」我没有直接验证**,这是推断。
- **[假设]** 卡的根因 = WiFi 丢包/抖动 + 重传缺失。
  → WiFi 抖动是事实(1.2),"重传缺失导致卡"是推断。
- **[观察,原因待查]** 把 PipeWire 强制改成 `ALAC + 不加密` 后,自愈能力变差、更频繁卡;回退到默认 `PCM + fp_sap25` 后自愈恢复。
  → 这是用户实测到的**现象**,但**为什么会这样,原因未明**(可能是 PipeWire 对 none/ALAC 路径实现较差,而非协议本身的问题)。
  → 注意:这**不代表** Go 版不能走"不加密"——AirMusic 正是不加密且不卡,说明协议层不加密是可行的;PipeWire 改 none 变差是 PipeWire 的实现问题。

---

## 三、未知 / 待逆向(还没搞清,**不要当事实用**)

需要从 `~/Desktop/airmusic/captures/am_cap.pcap`(AirMusic 完整抓包,含音频/控制流)逐包逆出来:

- [ ] **RTP 音频包的确切字节布局**(头各字段、payload 起始)— 待核对
- [ ] **重传请求 / 响应包的确切格式**(control 端口)— ⚠️ 最关键的缺口,Go 要靠它防卡
- [ ] **sync 同步包 / timing 对时包的格式**
- [ ] ALAC 未压缩帧的封装细节(若用 ALAC)

---

## 四、Go 实现计划

### 4.1 架构(goroutines)
```
main: mDNS 发现 Mac(_raop._tcp)→ RTSP 握手(TCP)
 ├─ A 抓系统音频(parec/pw-record monitor → 编码)→ 发送队列
 ├─ B 按节奏打 RTP 包发 UDP 到 server_port,同时存入"已发送环形缓冲"
 ├─ C 监听 control 端口,收到重传请求 → 从环形缓冲补发  ← 防卡核心
 ├─ D 周期发 sync 包;响应 timing 端口对时
 └─ E RTSP keepalive + 连接健康检测,僵死/断开就自动重做握手
```

### 4.2 关键设计点
- **重传(C)** 是和 PipeWire 拉开差距的核心,必须做对(但格式还没逆出来,见 §三)
- **不加密**:走 et=0(依据 AirMusic 抓包),跳过 FairPlay/fp_sap25,大幅降低实现难度
- **连接自愈(E)**:检测到僵死(类似 §1.3 的"假 RUNNING")就自动重连,不像 PipeWire 卡死不动
- 延迟可配:RTP 时间戳提前量。**注意:大缓冲是否有益,取决于重传是否做好;在重传验证可用之前不要盲目调大**(此点为设计权衡,非实测结论)

### 4.3 里程碑
- **M1 最小可用**:mDNS 发现 + 握手(不加密,照 §1.1)+ PCM 音频 RTP 发送 + 基础重传 → 跟 PipeWire 跑同一首歌对照,验证更稳
- **M2**:完善 sync/timing 同步 + 自动重连
- **M3**:ALAC 编码 + 延迟可调
- **M4**:UI(设备列表 / 连接 / 状态 / 延迟)

### 4.4 新对话的第一步
1. 用 tshark 打开 `~/Desktop/airmusic/captures/am_cap.pcap`,**逆出 control 端口的重传请求/响应包格式**(填补 §三 最关键的缺口)
2. 搭 Go 骨架,实现 M1
3. 对照 PipeWire 验证

---

## 资产
- `captures/am_ok_stream.txt` — AirMusic 成功握手的完整 RTSP 文本(本仓库已含)
- 以下大文件**未入库**,在本地 `~/Desktop/airmusic/`:
  - `captures/am_cap.pcap` — AirMusic 完整抓包(含音频/控制,逆 §三 用)
  - `base.apk` + `out2/` — AirMusic 反编译(核心 `out2/sources/app/airmusic/sinks/airplay/b.java`)
  - `caster.py` — Python 原型(握手能通、认证过,但无重传 → 会卡;可参考握手代码)

## 参考实现(读它们的重传/同步逻辑)
- shairport-sync(C,接收端)
- owntone 的 `raop_device.c`(C,发送端)
- pyatv(Python,RAOP 客户端)

## 协议名词
- RAOP = AirPlay 1 音频协议;RTSP 信令(TCP 7000)+ RTP 音频(UDP)
- approve = Mac 弹的"是否允许连接",**在 Mac 上点**,通常首次/重连时出现
