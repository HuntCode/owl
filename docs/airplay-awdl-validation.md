# AirPlay over OWL/AWDL Validation

Date: 2026-04-28

目标环境：

- Raspberry Pi 5 作为 Linux 接收端主机。
- OWL 负责通过 `awdl0` 提供 AWDL 网络能力。
- UxPlay 或其他 AirPlay 接收端负责投屏协议和渲染。
- 不假设 Raspberry Pi 5 板载 Wi-Fi 能满足 OWL 需要的 active monitor 和注入能力。
- 当前重点验证外接半高卡是否能作为 AWDL 无线网卡。购买时型号标注为 RTL8852BE，但当前系统实际枚举为 Atheros AR928X/AR9280。

读者背景：

- 你熟悉普通同 Wi-Fi 下的 AirPlay 接收端开发，包括 mDNS/DNS-SD。
- 你不一定熟悉 Wi-Fi 底层，所以本文会把 OWL 里的无线术语翻译成 AirPlay 接收端视角。
- 当前目标是借助 OWL/AWDL 实现“不在同一个 Wi-Fi 路由器下也能 AirPlay 投屏”，产品体验上接近经典 Miracast。

## AirPlay 视角下的目标

普通同 Wi-Fi 下的 AirPlay 流程大致是：

```text
iPhone/macOS
  -> 在局域网里发 mDNS 查询
  -> 接收端发布 _airplay._tcp / _raop._tcp
  -> 客户端连接接收端的 IP 和端口
  -> 接收端处理 AirPlay 控制流和媒体流
  -> 接收端渲染视频/音频
```

本项目希望变成：

```text
iPhone/macOS
  -> 唤醒 AWDL
  -> 通过 AWDL 发现 Raspberry Pi
  -> 在 awdl0 这条链路上通过 mDNS/DNS-SD 看到 UxPlay
  -> 通过 AWDL 的 IPv6 链路连接 UxPlay
  -> Pi 渲染视频/音频
```

所以核心心智模型是：**用 AWDL 点对点链路替代普通共享 Wi-Fi 局域网**，同时尽量让 AirPlay 接收端仍然按普通局域网程序来跑。

这在产品体验上类似 Miracast：用户不需要加入同一个路由器 Wi-Fi。底层实现则不同：Miracast 通常基于 Wi-Fi Direct，而这里依赖 Apple 的 AWDL、Bonjour/mDNS 和 AirPlay 协议。

## 模块分工

- OWL：参与 AWDL 网络，并在 Linux 里暴露一个 `awdl0` 网络接口。
- `awdl0`：Avahi/UxPlay 应该把它当成“AirPlay 局域网接口”。
- Avahi：在 `awdl0` 上发布和响应 Bonjour/DNS-SD。
- UxPlay：实现 AirPlay 接收端协议和渲染。
- 外接 Wi-Fi 卡：真正发/收底层 AWDL Wi-Fi 帧的物理无线网卡。当前系统识别为 AR928X/AR9280，驱动为 `ath9k`，接口为 `wlan1`。

最重要的边界是：**OWL 不实现 AirPlay，UxPlay 不实现 AWDL**。它们之间的集成点就是 Linux 网络接口 `awdl0`。

## 常见 Wi-Fi/AWDL 术语

- AWDL：Apple Wireless Direct Link。Apple 的点对点无线链路，用于 AirDrop、Sidecar、部分 AirPlay 场景和附近设备能力。
- `awdl0`：OWL 创建的虚拟 TAP 网卡。应用在这里看到的是 Ethernet/IPv6 帧；OWL 负责把它们转换成真实 Wi-Fi 网卡上的 AWDL 帧。
- Monitor mode：监听模式。网卡不再作为普通 Wi-Fi 客户端连接 AP，而是接收原始 802.11 无线帧。
- Active monitor mode：主动监听模式。比普通 monitor mode 多一个关键能力：网卡/驱动会对发给自己的帧做 Wi-Fi 层 ACK。OWL 很需要这个能力。
- Frame injection：帧注入。发送自定义的原始 Wi-Fi 帧。AWDL 不是普通 managed Wi-Fi 流量，所以 OWL 必须能注入帧。
- Radiotap：Linux/libpcap 处理原始 Wi-Fi 帧时使用的元数据头，里面包含速率、信号强度、频道、flags 等信息。
- ACK：Wi-Fi MAC 层确认包，发生在 IP/TCP/UDP 下面。这个能力如果缺失，AirPlay 代码和 mDNS 都写对了也可能不稳定。
- Channel：Wi-Fi 频道。此 OWL 项目只直接支持 6、44、149。
- Social channels：AWDL 常用的发现/通信频道。在这里可以先理解成 6、44、149。
- Regulatory domain：国家/地区无线法规设置。如果频道标记为 `disabled` 或 `no IR`，Linux 可能禁止在该频道主动发射。
- Availability window：AWDL 的可用时间窗口。AWDL 不是一直在每个频道上通信，而是按时间窗口切换和收发，所以 mDNS 在 AWDL 上可能表现为一阵一阵的。
- PSF/MIF：AWDL 的管理/action 帧，用来广播存在、同步时间、频道计划和 peer 信息。验证时可以先把它理解成“OWL/AWDL 活着，并且正在发现附近 Apple 设备”。
- Peer：另一个 AWDL 参与者，比如 iPhone 或 Mac。
- IPv6 link-local：`fe80::` 开头的链路本地 IPv6 地址。AWDL 通常靠这种地址通信，所以命令里经常需要指定接口，例如 `-I awdl0`。

## 工作假设

推荐验证的架构是：

```text
iPhone/macOS AirPlay client
        |
      AWDL
        |
AR928X/ath9k wlan1 in active monitor mode
        |
       owl
        |
      awdl0
        |
Avahi DNS-SD + UxPlay
```

OWL 不是 AirPlay 接收端。它创建一个 Linux 虚拟网络接口承载 AWDL 流量，让普通支持 IPv6 的程序可以使用这个接口。UxPlay、Avahi、防火墙和渲染器仍然需要把 `awdl0` 当成可用的本地网络接口。

主要可行性问题不是“这张卡能不能正常连 Wi-Fi”，而是你的 Pi 5 内核、无线驱动、固件、监管域和天线配置能不能满足 OWL 需要的组合能力：

- active monitor mode
- radiotap capture
- frame injection
- 足够可靠的 Wi-Fi 层 ACK 行为
- 能稳定工作在 OWL 支持的频道之一：6、44、149

## 当前实测状态

最近一次硬件枚举和能力检查结果：

- `lspci -nnk` 显示外接 PCIe Wi-Fi 卡是 `Qualcomm Atheros AR928X Wireless Network Adapter [168c:002a]`。
- 内核日志显示具体为 `Atheros AR9280 Rev:2`。
- 当前驱动为 `ath9k`，接口为 `wlan1`，对应 `phy1`。
- 购买时标注的 RTL8852BE 没有出现在 PCIe 枚举里；如果外观/商品信息写 RTL8852BE，实际芯片或当前插入卡与商品标注不一致。
- 加入 `dtoverlay=pcie-32bit-dma-pi5` 后，之前的 `ath9k ... Failed to allocate tx descriptors: -12` 已解决。
- `wlan1` 不承载默认路由；当前普通联网走 `wlan0`。

`phy1` 能力结论：

- 支持 `monitor` interface mode。
- `iw phy phy1 info` 明确显示 `Device supports active monitor (which will ACK incoming frames)`。
- 支持 `set_channel` 和 `frame` 命令。
- 支持 TX/RX 多种管理帧类型，满足后续 frame injection 验证的基础条件。
- 当前监管域下 channel 6 可用；channel 44 和 149 在 `phy1` 上显示 `no IR`，所以第一轮 OWL 验证建议从 channel 6 开始。

已做过的临时 active monitor 冒烟测试：

```sh
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo iw dev wlan1 set monitor active
sudo ip link set wlan1 up
sudo iw dev wlan1 set channel 6 HT20
iw dev wlan1 info
```

结果：

- `set type monitor` 成功。
- `set monitor active` 成功。
- `set channel 6 HT20` 成功。
- `iw dev wlan1 info` 显示 `type monitor`、`channel 6 (2437 MHz)`。
- 测试后已把 `wlan1` 恢复为 managed。

### 2026-04-28 运行验证结果

本轮已经安装了基础验证依赖：

- `tcpdump`
- `avahi-utils`
- `libpcap-dev`
- `libev-dev`
- `libnl-3-dev`
- `libnl-genl-3-dev`
- `libnl-route-3-dev`

OWL 主程序构建结果：

- `cmake -S . -B build` 成功。
- `cmake --build build --target owl -j1` 成功，生成 `build/daemon/owl`。
- 完整 `cmake --build build` 会在 bundled GoogleTest 上失败，原因是当前编译器把 `gtest-death-test.cc` 的 `maybe-uninitialized` 警告当作错误；这不影响 `daemon/owl` 主程序验证。

OWL channel 6 启动结果：

- 直接运行 `sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 6 -vv` 失败，关键错误是 `Object busy`，发生在 OWL 自动把 `wlan1` 切到 active monitor mode 时。
- 手动执行 `ip link set wlan1 down`、`iw dev wlan1 set type monitor`、`iw dev wlan1 set monitor active`、`ip link set wlan1 up`、`iw dev wlan1 set channel 6 HT20` 后，再运行 `sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 6 -vv -N` 成功。
- 成功启动后，OWL 日志显示 `WLAN device: wlan1`、`Host device: awdl0`、`Channel 6 [2437 MHz] is available for frame injection`，并持续发送 MIF/PSF。
- `awdl0` 被创建，状态为 UP，MTU 为 1450，MAC 与 `wlan1` 一致。
- `awdl0` 获得 IPv6 link-local 地址：`fe80::b674:9fff:fe6c:bc2f/64`。

抓包结果：

- `tcpdump -i wlan1` 可以以 `IEEE802_11_RADIO`/radiotap 方式抓包。
- 在 channel 6 上能看到 OWL 自己发出的 AWDL vendor action frame，源 MAC 为 `b4:74:9f:6c:bc:2f`，形如 `Action: Vendor Act#0`。
- 45 秒内过滤掉本机源 MAC 后，没有抓到外部 AWDL action frame。
- 同一时间 `tcpdump -i awdl0 ip6` 没有抓到 IPv6 包。
- `ip -6 neigh show dev awdl0` 没有出现 peer。

channel 44/149 结果：

- `iw phy phy1 channels` 仍显示 channel 44 和 149 为 `No IR`。
- `sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 44 -vv -N` 会启动，但日志提示 `Channel 44 [5220 MHz] does not allow to initiate radiation first (no IR)`。
- `sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 149 -vv -N` 同样提示 `Channel 149 [5745 MHz] does not allow to initiate radiation first (no IR)`。
- 因此当前 AR9280/ath9k 在这台 Pi 5 上，5 GHz 的两个 AWDL 常用频道不适合作为首选验证路径。

本轮结束状态：

- 已停止 OWL。
- `awdl0` 已消失。
- `wlan1` 已恢复为 managed。
- NetworkManager 中 `wlan1` 是 disconnected，默认联网仍走 `wlan0`。
- `UxPlay` 当前未安装；Debian trixie apt 源里可选版本为 `1.71.1-1`。

当前结论：

- AR9280/ath9k 的 active monitor 基础能力可用。
- `tcpdump` radiotap 抓包可用。
- OWL 在 channel 6 上可以通过手动 active monitor + `-N` 路径创建 `awdl0`。
- OWL 自己的 AWDL action frame 能在本机 monitor 抓包中看到，说明发送路径至少进入了无线接口。
- 尚未证明 iPhone 和 OWL 互相发现，也尚未证明 `awdl0` 上出现来自 Apple peer 的 IPv6/mDNS 流量。
- 下一步重点不是 UxPlay，而是先让 iPhone peer 出现在 OWL 日志或 `ip -6 neigh show dev awdl0` 里。

### 2026-04-28 追加 peer 发现验证

本轮继续推进 iPhone peer 发现层，重点排除 RSSI 过滤和 tcpdump 过滤表达式误判。

`-f` 模式结果：

- 使用手动 active monitor 后启动 `sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 6 -vv -f -N` 成功。
- `-f` 表示关闭 OWL 的 RSSI 过滤。
- `awdl0` 仍能创建，并获得同一个 IPv6 link-local 地址。
- 60 秒抓包里，宽泛 action-frame 过滤能看到周围普通 Wi-Fi 的 Block Ack/Reserved action 帧，但这些不是 AWDL。
- 收窄到 vendor-specific action frame 后，外部帧为 0：

```sh
sudo tcpdump -i wlan1 -e -n -vv \
  'wlan[0] == 0xd0 and wlan[24] == 0x7f and not wlan addr2 b4:74:9f:6c:bc:2f'
```

- 同期 `awdl0` 上只看到本机发出的 IPv6 router solicitation 和 mDNS 查询，没有外部 peer IPv6 包。
- `ip -6 neigh show dev awdl0` 仍为空。

tcpdump 过滤表达式自校验：

- 再次短暂启动 OWL，使用下面的过滤条件可以抓到本机发出的 AWDL vendor action frame：

```sh
sudo tcpdump -i wlan1 -e -n -vv \
  'wlan[0] == 0xd0 and wlan[24] == 0x7f and wlan addr2 b4:74:9f:6c:bc:2f'
```

- 这说明 `wlan[24] == 0x7f` 这个 vendor-specific action 过滤条件是有效的。

纯监听 6/44/149 结果：

- 停止 OWL 后，将 `wlan1` 作为纯 monitor sniffing 接口。
- 分别固定到 channel 6、44、149，每个频道监听约 25 秒。
- 三个频道都没有看到外部 vendor-specific action frame。
- 因此当前没有观察到 iPhone 在 6/44/149 任一固定社交信道上发出可见 AWDL action frame。

本轮结束状态：

- 已停止 OWL。
- 已停止所有 tcpdump。
- `awdl0` 已消失。
- `wlan1` 已恢复为 managed。
- NetworkManager 中 `wlan1` 是 disconnected，默认联网仍走 `wlan0`。

新的判断：

- `-f` 不能解决 peer 发现问题。
- 当前最可能的下一步不是 Avahi/UxPlay，而是继续确认 iPhone 是否真的进入 AWDL 发现状态，以及 AR9280/ath9k 是否能在实际 Apple AWDL 场景里看到对端。
- 下次重测时建议明确使用 iPhone 的 AirDrop/隔空投送触发：打开照片或文件的分享面板，进入 AirDrop 页面，接收设置临时设为“所有人/Everyone for 10 Minutes”，让手机靠近树莓派并保持屏幕点亮。
- 如果这样仍然在 6/44/149 都看不到外部 vendor action frame，建议引入第二块已知能抓 Apple AWDL 的监听设备或换一张已知 OWL 兼容的 Wi-Fi 卡复测，用来区分 iPhone 触发问题和当前 AR9280/ath9k/转接板组合问题。

## 项目代码结论

### OWL 提供的是 IPv6 AWDL 链路

README 里说明，OWL 通过创建虚拟网络接口集成进 Linux 网络栈，让已有的 IPv6 程序可以直接使用 AWDL。启动后会创建 `awdl0`，并给它配置 link-local IPv6 地址；发现 AWDL peer 后，会把 peer 加入系统 neighbor table。

相关本地代码：

- `README.md`：说明 active monitor mode、频道 6/44/149、`awdl0` 和 IPv6 neighbor。
- `daemon/io.c`：创建 TAP 接口，拉起接口，把它设置成和 WLAN 接口相同的 MAC 地址，并设置 MTU 1450。
- `daemon/core.c`：从 `awdl0` 读取 Ethernet 帧，区分 multicast/unicast 队列，发送/接收 AWDL 帧，并为 peer 添加/删除 IPv6 neighbor。
- `src/tx.c`：把主机侧 payload 包装成 AWDL data frame。当前发送路径里的 AWDL data ethertype 是 IPv6。
- `src/rx.c`：把收到的 AWDL data frame 解包回 Ethernet 帧，并写入 `awdl0`。

重要含义：先把 `awdl0` 当成 IPv6-only 的验证目标。不要一开始期待 IPv4 AirPlay discovery 或 IPv4 AirPlay control 能直接跑在 OWL 上，因为当前 OWL 发送路径把 AWDL data ethertype 固定成 IPv6。

### OWL 会独占这个无线接口

在 Linux 上，OWL 会先把指定 WLAN 接口 down 掉，通过 nl80211 把它改成 active monitor mode，再 up 起来，然后用 libpcap 打开它。当前代码不会创建一个额外的 monitor interface。

相关本地代码：

- `daemon/netutils.c`：`set_monitor_mode()` 设置 `NL80211_IFTYPE_MONITOR` 和 `NL80211_MNTR_FLAG_ACTIVE`。
- `daemon/io.c`：`io_state_init_wlan()` 会直接修改传入的无线接口。
- `README.md`：当前限制说明 OWL 不允许同一个 Wi-Fi 接口同时连接 AP。

实际含义：测试时给 OWL 用的外接 Wi-Fi 接口不应该再被 NetworkManager/wpa_supplicant 管理。当前实测中这个接口是 `wlan1`。普通网络连接建议走以太网或另一块 Wi-Fi。

### AirPlay 发现强依赖 multicast/mDNS

AirPlay 发现通常依赖 Bonjour/DNS-SD，也就是 UDP 5353 上的 mDNS。在 OWL 里，从 `awdl0` 出来的 multicast 帧会被单独排队，并且只在 AWDL multicast availability window 里发送。

相关本地代码：

- `daemon/core.c`：通过 Ethernet 目的 MAC 的 multicast bit 判断 multicast 帧，并放入 `tx_queue_multicast`。
- `src/schedule.c`：只允许在 AWDL slot 0 和 10 发送 multicast。

实际含义：mDNS 在无线侧看起来可能不是连续的，而是按 AWDL 窗口一阵一阵地发。判断 Avahi 是否发送前，建议同时抓 `awdl0` 和无线侧。

## 外部资料结论

UxPlay v1.73 文档说明，它是 AirPlay 镜像/音频接收端，并且已经在 Raspberry Pi 5 上测试过。它依赖本地网络上的 DNS-SD 服务发现，Linux 上通常由 Avahi 提供。UxPlay 也有较新的 Bluetooth LE beacon 发现方式，但这个 beacon 广播的是 IPv4 地址。因为当前 OWL 发送路径更适合先按 IPv6 验证，所以纯 AWDL 路径不建议从 BLE beacon 入手。

Linux kernel 文档把 active monitor 定义为一种 monitor flag：它会 ACK 发到本接口 MAC 地址的帧。OWL 明确请求了这个 flag。独立的 Wi-Fi injection 测试资料也提醒：active monitor 支持很有限，必须用实际驱动、实际网卡、实际内核验证，不能从“普通 Wi-Fi 能用”推断出来。

购买时标注的 RTL8852BE 在现代 Linux 内核上通常由 `rtw89_8852be` 驱动支持，但当前系统没有枚举到 Realtek 设备，实际需要验证的是 `ath9k` 驱动下的 AR928X/AR9280。

## 推荐验证策略

因为你已经熟悉 AirPlay 接收端，建议把问题拆成五个独立问题：

1. 外接 Wi-Fi 卡能不能完成 OWL 需要的底层 Wi-Fi 工作？
2. OWL 能不能创建可用的 `awdl0`，并发现 Apple AWDL peer？
3. 普通 IPv6 流量能不能走通 `awdl0`？
4. Avahi 能不能明确地在 `awdl0` 上发布 AirPlay DNS-SD 记录？
5. UxPlay 能不能接受来自 IPv6 link-local 客户端的 AirPlay 控制流和媒体流？

这个顺序很重要。如果第 1 或第 2 步失败，改 UxPlay 或 mDNS 代码基本帮不上忙。如果第 3 步成功但第 4 步失败，问题大概率在 Avahi 的接口绑定或服务发布。如果第 4 步成功但第 5 步失败，问题大概率在 UxPlay 的 IPv6 监听、防火墙端口，或 AirPlay 协议对 link-local 地址的兼容性。

测试时要避免假阳性：

- Apple 客户端和 Pi 尽量不要同时处在同一个普通 Wi-Fi 局域网里，或者至少抓包证明 AirPlay 走的是 `awdl0`。
- 不能只因为“UxPlay 出现在 Screen Mirroring 列表里”就认为 AWDL 成功，必须用 `tcpdump -ni awdl0` 看到 mDNS 和连接流量。
- 不要一开始就走 BLE beacon。UxPlay 的 BLE beacon 广播 IPv4 地址，而当前 OWL 代码的发送路径更适合先按 IPv6 验证。
- 先从 channel 6 开始。基础链路打通后，再测试 44 或 149。

对你的接收端实现来说，第一个不改代码的目标应该是：

```text
Avahi publishes _airplay._tcp / _raop._tcp on awdl0
UxPlay listens on IPv6
iPhone connects to UxPlay over fe80::...%awdl0
```

## 验证点总览

下面这些验证点按从底层无线能力到 AirPlay 投屏的顺序排列。后续排查时建议按顺序推进，前一层没有通过时，先不要改后一层的 AirPlay 或 mDNS 逻辑。

### 1. 无线网卡能力

确认外接 Wi-Fi 卡是否能满足 OWL 的底层要求。

验证点：

- 能进入 monitor mode。
- 能进入 active monitor mode。
- 能抓到 radiotap 格式的 802.11 帧。
- 能做 frame injection。
- 注入的帧真的能在无线侧被观察到。
- ACK/重试行为足够稳定。

如果这里不通，后面的 mDNS、Avahi、UxPlay 验证都不会真正成立。

### 2. OWL 是否正常启动

确认 OWL 能使用外接 Wi-Fi 卡创建 AWDL 节点。

验证点：

- `sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 6 -vv` 能启动，或者在本机当前环境下使用手动 active monitor 后的 `-N` 路径启动。
- `awdl0` 被创建。
- `awdl0` 是 UP 状态。
- `awdl0` 有 IPv6 link-local 地址。
- OWL 没有报 monitor、channel、pcap、injection 相关错误。

本机当前实测要点：

- 自动 monitor 切换会遇到 `Object busy`。
- 可用 workaround 是先手动执行 `iw dev wlan1 set type monitor` 和 `iw dev wlan1 set monitor active`，再用 `-N` 启动 OWL。
- channel 6 可用于 frame injection；channel 44/149 当前有 `No IR` 警告。

### 3. iPhone 是否发现树莓派这个 AWDL peer

确认 iPhone 和树莓派进入同一个 AWDL 近场通信环境。

验证点：

- iPhone 开启 Wi-Fi 和 Bluetooth。
- 打开 AirDrop 或 Screen Mirroring 后，OWL 日志能看到 AWDL action frame。
- OWL 日志出现 `add peer ...`。
- `ip -6 neigh show dev awdl0` 能看到 peer。
- peer 不频繁 add/remove。

这一步验证的是：iPhone 和 OWL 是否已经在 AWDL 层互相看见。

### 4. `awdl0` 上是否能跑 IPv6

确认 AWDL 链路已经映射成 Linux 可用网络接口。

验证点：

- `ip -6 addr show dev awdl0` 有 `fe80::...`。
- `ip -6 neigh show dev awdl0` 有 iPhone/Mac peer。
- `tcpdump -ni awdl0` 能看到 IPv6 包。
- 如果拿到 peer 地址，可以测试 `ping -6 -I awdl0 fe80::<peer>`。

这一步验证的是：`awdl0` 是否已经像一张点对点局域网网卡。

### 5. mDNS 是否走 `awdl0`

确认 AirPlay 发现流量真的跑在 AWDL 上，而不是普通 Wi-Fi 上。

验证点：

- Avahi 在 `awdl0` 上启用。
- `tcpdump -ni awdl0 'udp port 5353'` 能看到 mDNS query/response。
- iPhone 打开 Screen Mirroring 时，`awdl0` 上能看到 mDNS 查询。
- Avahi 能在 `awdl0` 上响应 `_airplay._tcp` / `_raop._tcp`。
- 断开普通同 Wi-Fi 后，仍能看到这些流量。

### 6. AirPlay 服务记录是否正确

确认 iPhone 看到的是可连接的 UxPlay 服务。

验证点：

- `avahi-browse -a -r` 能看到 UxPlay 服务。
- 服务出现在 `awdl0` 接口上。
- 服务记录里有可用 IPv6 地址。
- TXT record 符合 AirPlay 客户端预期。
- 服务端口和 UxPlay 实际监听端口一致。

### 7. UxPlay 是否监听 `awdl0` 可达地址

确认发现之后真的能连接。

验证点：

- `ss -tulpen` 能看到 UxPlay 监听 IPv6。
- UxPlay 没有只绑定 `127.0.0.1` 或普通 LAN IPv4。
- 防火墙允许 UxPlay 端口。
- iPhone 选择投屏时，`tcpdump -ni awdl0` 能看到 TCP 连接尝试。
- UxPlay 日志能看到 RTSP/pairing 请求。

### 8. AirPlay 控制链路是否成功

确认发现和连接后，协议协商能通过。

验证点：

- iPhone 选择 UxPlay 后不立即失败。
- UxPlay 日志能看到 pairing/setup。
- RTSP 交互正常。
- 没有地址、端口、认证、加密协商错误。

### 9. 媒体流是否成功

确认投屏真正开始。

验证点：

- UxPlay 日志显示 media session 启动。
- `awdl0` 上能看到持续 UDP/TCP 媒体流。
- Pi 上有画面/声音。
- 延迟、卡顿、断流在可接受范围内。

### 10. 稳定性

确认不是偶然跑通。

验证点：

- 连续投屏 5-10 分钟不断。
- iPhone 熄屏/亮屏后行为可预期。
- 断开后能重新发现和连接。
- peer 不频繁消失。
- OWL 日志里没有大量 unknown frame、retry、channel 失败。
- 换距离、换频道后知道边界在哪里。

关键里程碑可以压缩成五个：

1. 外接 Wi-Fi 卡能 active monitor + injection。
2. OWL 能创建 `awdl0` 并发现 iPhone peer。
3. `awdl0` 上能看到 IPv6/mDNS。
4. UxPlay 服务能通过 `awdl0` 被 iPhone 发现。
5. iPhone 能通过 `awdl0` 连接 UxPlay 并开始媒体流。

## 验证清单

### 1. 系统和硬件基线

在改变接口模式前先记录：

```sh
uname -a
cat /etc/os-release
iw --version
ip link
iw dev
lspci -nnk
dmesg | grep -Ei 'ath|ath9k|168c|firmware|cfg80211|tx descriptors'
```

通过标准：

- 外接 Wi-Fi 卡出现在 PCIe 设备列表里。当前实测应为 `Qualcomm Atheros AR928X [168c:002a]`。
- 绑定的驱动是当前内核期望的 `ath9k`。
- 固件加载没有明显报错。
- OWL 测试接口能和 Pi 5 板载 Wi-Fi 明确区分。

### 2. 让外接 Wi-Fi 卡脱离网络管理器

测试期间让外接 Wi-Fi 卡只服务于 OWL。当前实测接口名是 `wlan1`：

```sh
nmcli device status
sudo nmcli dev set wlan1 managed no
sudo rfkill unblock wifi
```

通过标准：

- 没有 wpa_supplicant/NetworkManager 进程立刻重新配置 `wlan1`。
- 如果需要普通联网，使用以太网或另一块 Wi-Fi。

### 3. 检查 monitor 和 active monitor 能力

查看 PHY 能力：

```sh
iw phy phyX info
iw list
```

重点看：

- `iw list` 的 supported interface modes 列表里有 `monitor`。
- 如果工具会显示 active-monitor feature，确认有 `active monitor`。
- 频道 6、44 或 149 没有被标记为 `disabled` 或 `no IR`。

手动 active monitor 冒烟测试：

```sh
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo iw dev wlan1 set monitor active
sudo ip link set wlan1 up
iw dev wlan1 info
sudo tcpdump -I -i wlan1 -y IEEE802_11_RADIO -vv
```

通过标准：

- `set monitor active` 成功。
- tcpdump 能以 radiotap 格式打开设备。
- 在选定频道上能看到附近 Wi-Fi 帧。

失败现象：

- active monitor 返回 `Operation not supported`。
- tcpdump 不能使用 `IEEE802_11_RADIO`。
- 接口进入 monitor mode 但在已知繁忙频道上看不到帧。

### 4. 测试注入和 ACK 行为

推荐用第二块支持 monitor 的网卡做测试：

```sh
git clone https://github.com/vanhoefm/wifi-injection.git --recursive
cd wifi-injection
./pysetup.sh
sudo su
source venv/bin/activate
./test-injection.py wlan1 <second_monitor_iface> --active --channel 6
```

如果没有第二块网卡，可以先做最小 OWL 方向测试：

```sh
sudo nmcli dev set wlan1 managed no
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo iw dev wlan1 set monitor active
sudo ip link set wlan1 up
sudo iw dev wlan1 set channel 6 HT20
sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 6 -vv -N
```

然后用附近 Apple 设备打开 AirDrop 或 Screen Mirroring，触发 AWDL。

通过标准：

- injection test 的关键项目通过。
- OWL 日志里能看到反复发送 PSF/MIF，并能收到 AWDL action frame。
- OWL 最终打印 `add peer ...`。
- `ip -6 neigh show dev awdl0` 能看到 AWDL peer。

失败现象：

- pcap 显示帧已发送，但无线侧完全抓不到。
- 能发现 peer，但 unicast data 始终不成功。
- 大量重试或 peer 频繁 add/remove，通常暗示 ACK 缺失或频道不同步。

### 5. 验证监管域和频道选择

先从 channel 6 开始，因为它能避开很多 5 GHz 监管域和驱动问题：

```sh
iw reg get
iw list
sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 6 -vv
```

channel 6 跑通后，再重复测试 44 或 149。

通过标准：

- 选定频道没有被标记为 `disabled` 或 `no IR`。
- OWL 没有警告该频道不能注入。
- 换频道后仍能发现 Apple peer。

### 6. 验证 `awdl0`

OWL 运行时检查：

```sh
ip link show awdl0
ip -6 addr show dev awdl0
ip -6 route show dev awdl0
ip -6 neigh show dev awdl0
sudo tcpdump -ni awdl0
```

如果出现 peer 的 link-local 地址：

```sh
ping -6 -I awdl0 fe80::<peer-address>
```

通过标准：

- `awdl0` 是 UP 状态。
- `awdl0` 有 link-local IPv6 地址。
- IPv6 neighbor table 里出现 peer。
- link-local ping 成功，或者至少能在 OWL 日志里看到 unicast 尝试。

### 7. 先证明 UxPlay 在普通局域网可用

混入 AWDL 前，先验证接收端栈本身：

```sh
sudo systemctl status avahi-daemon
uxplay -h
uxplay -n OwlPi -p 7100 -vsync no
avahi-browse -a -t -r
ss -tulpen
```

通过标准：

- 在普通 LAN 下，UxPlay 能出现在 iOS/macOS Screen Mirroring 列表里。
- Pi 5 上音视频渲染正常。
- 防火墙允许 mDNS 以及 UxPlay 需要的 TCP/UDP 端口。
- `ss` 能确认 UxPlay 监听了 IPv6，或者至少确认它实际绑定了哪些地址。

这个基线没跑通前，不要进入 AWDL 验证。

### 8. 将 Avahi 和 UxPlay 绑定到 `awdl0`

为了做纯 AWDL 测试，先让发现路径尽量明确。可以在 `/etc/avahi/avahi-daemon.conf` 里测试类似配置：

```ini
[server]
use-ipv6=yes
use-ipv4=no
allow-interfaces=awdl0

[publish]
disable-publishing=no
```

等 OWL 创建 `awdl0` 后重启：

```sh
sudo systemctl restart avahi-daemon
avahi-browse -a -t -r
sudo tcpdump -ni awdl0 'udp port 5353'
uxplay -n OwlPi-AWDL -p 7100 -vsync no
```

通过标准：

- `avahi-browse` 显示 UxPlay 服务发布在 `awdl0` 上，并且有 IPv6 地址。
- `awdl0` 上能看到 mDNS query 和 response。
- Avahi 响应时，OWL 的 multicast TX 计数或日志有增长。

失败现象：

- Avahi 只发布在 `lo` 或普通 LAN 接口上。
- UxPlay 只发布 IPv4 记录。
- mDNS 已经离开 `awdl0`，但看不到对应的 AWDL multicast 活动。

### 9. Apple 客户端测试阶梯

使用一台近距离 iPhone/iPad/Mac。客户端保持 Wi-Fi 和 Bluetooth 开启。每次测试都记录 OWL log、`awdl0` 上的 `tcpdump` 和 UxPlay log。

验证阶梯：

1. 打开 AirDrop 或 Screen Mirroring 后，OWL 能收到 AWDL action frame。
2. OWL 日志出现有效 peer，并添加 IPv6 neighbor。
3. 客户端通过 AWDL 发送 mDNS query，且 `awdl0` 上可见。
4. Avahi 在 `awdl0` 上响应。
5. UxPlay 出现在 Screen Mirroring 列表里。
6. 选择 UxPlay 后，客户端从自己的 link-local IPv6 地址发起 TCP 连接。
7. pairing/control 成功。
8. RTP/media 流启动，视频开始渲染。

在第一个失败步骤停下，按下面的故障分类定位。

### 10. 故障分类

没有 OWL peer：

- 重新检查 active monitor mode 和 frame injection。
- 先测试 channel 6。
- 把 Apple 设备放在 1-2 米内。
- 近距离测试时用 `-f` 运行 OWL，临时关闭 RSSI 过滤。
- 确认 OWL 用的是外接 Wi-Fi 卡 `wlan1`，而不是 Pi 板载 Wi-Fi `wlan0`。

有 peer 但没有 IPv6 流量：

- 检查 `ip -6 neigh show dev awdl0`。
- 同时抓 `awdl0` 和无线侧流量。
- 确认 peer 是有效状态：OWL 只有在 MIF、version、device class 信息都满足后才会 add peer。

`awdl0` 上能看到 mDNS，但接收端不出现：

- 确认 Avahi 在 `awdl0` 上发布 UxPlay 服务，而不只是 `lo`。
- 确认 UxPlay 的 DNS-SD 记录包含可用 IPv6 地址。
- 检查 Apple 客户端是否仍然通过普通 LAN 发现服务，这会掩盖 AWDL-only 的问题。

接收端出现了，但连接失败：

- 用 `ss -tulpen` 检查 UxPlay 监听 socket。
- 在防火墙中放行 UxPlay 需要的 TCP/UDP 端口。
- 验证 UxPlay 接受 IPv6 link-local 客户端。
- 在 UxPlay 日志里关注 RTSP/pairing 错误。

连接能开始但不稳定：

- 优先怀疑 ACK/retry 行为或 AR928X/ath9k 注入质量。
- 如果正在用 44/149，先回到 channel 6。
- 用另一张已知可工作的 OWL 网卡复测，用来区分软件问题和当前外接卡/转接板问题。

## 决策点

只有下面这些都通过，才值得继续押注当前 AR928X/ath9k 路线：

- active monitor mode 可以启用；
- 注入帧能在无线侧被观察到；
- OWL 能发现 Apple peer；
- `awdl0` 能获得 IPv6 neighbor；
- `awdl0` 上能看到 link-local IPv6 流量。

如果这些通过但 UxPlay 不能被发现，重点看 Avahi/UxPlay 的 IPv6 DNS-SD 接口绑定。

如果 UxPlay 能被发现但连不上，重点看 UxPlay IPv6 监听、防火墙端口和 AirPlay 协议兼容性。

如果接收端路径强依赖 IPv4，这个不改代码的方案就不够。当前 OWL 发送路径把 payload 标成 IPv6，所以 IPv4 over AWDL 需要改 OWL 代码，或者换一条接收/发现策略。

## 参考资料

- 本仓库 OWL README：`README.md`
- UxPlay upstream README: https://github.com/FDH2/UxPlay
- Linux cfg80211 monitor flags documentation: https://docs.kernel.org/driver-api/80211/cfg80211.html
- Wi-Fi injection/active monitor testing notes: https://github.com/vanhoefm/wifi-injection
- Linux Kernel Driver Database entry for RTL8852BE: https://cateee.net/lkddb/web-lkddb/RTW89_8852BE.html
- Raspberry Pi forum thread on Atheros PCIe DMA failure: https://forums.raspberrypi.com/viewtopic.php?t=343846
- Launchpad bug about `pcie-32bit-dma` workaround: https://bugs.launchpad.net/bugs/1927037
