# AirPlay over OWL/AWDL Validation

Date: 2026-04-28
Last updated: 2026-04-29

目标环境：

- Raspberry Pi 5 作为 Linux 接收端主机。
- OWL 负责通过 `awdl0` 提供 AWDL 网络能力。
- UxPlay 或其他 AirPlay 接收端负责投屏协议和渲染。
- 不假设 Raspberry Pi 5 板载 Wi-Fi 能满足 OWL 需要的 active monitor 和注入能力。
- 当前重点验证外接半高卡是否能作为 AWDL 无线网卡。当前已插入并已测试的外接卡统一按 AR928X 记录；另有一张新购对比卡待到货。

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
- 外接 Wi-Fi 卡：真正发/收底层 AWDL Wi-Fi 帧的物理无线网卡。当前系统识别为 AR928X，驱动为 `ath9k`，接口为 `wlan1`；待到货对比卡为 AR9280。

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
- 内核日志曾显示 `Atheros AR9280 Rev:2`，但本文按当前实物/PCI 枚举口径记为“当前 AR928X”；待到货的新卡单独记为“AR9280”。
- 当前驱动为 `ath9k`，接口为 `wlan1`。最近一次 `iw dev` 中它对应 `phy0`；早先记录里可能出现过 `phy1`，后续命令以本机实时 `iw dev` 输出为准。
- 加入 `dtoverlay=pcie-32bit-dma-pi5` 后，之前的 `ath9k ... Failed to allocate tx descriptors: -12` 已解决。
- `wlan1` 不承载默认路由；当前普通联网走 `wlan0`。

`phyX` 能力结论：

- 支持 `monitor` interface mode。
- `iw phy phyX info` 明确显示 `Device supports active monitor (which will ACK incoming frames)`。
- 支持 `set_channel` 和 `frame` 命令。
- 支持 TX/RX 多种管理帧类型，满足后续 frame injection 验证的基础条件。
- 当前监管域下 channel 6 可用；channel 44 和 149 在该 phy 上显示 `no IR`，所以第一轮 OWL 验证建议从 channel 6 开始。

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

- `iw phy phyX channels` 仍显示 channel 44 和 149 为 `No IR`。
- `sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 44 -vv -N` 会启动，但日志提示 `Channel 44 [5220 MHz] does not allow to initiate radiation first (no IR)`。
- `sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 149 -vv -N` 同样提示 `Channel 149 [5745 MHz] does not allow to initiate radiation first (no IR)`。
- 因此当前 AR928X/ath9k 在这台 Pi 5 上，5 GHz 的两个 AWDL 常用频道不适合作为首选验证路径。

本轮结束状态：

- 已停止 OWL。
- `awdl0` 已消失。
- `wlan1` 已恢复为 managed。
- NetworkManager 中 `wlan1` 是 disconnected，默认联网仍走 `wlan0`。
- 当时 `UxPlay` 尚未安装；Debian trixie apt 源里可选版本为 `1.71.1-1`。后续 Avahi/UxPlay 阶段已经安装该版本。

当前结论：

- AR928X/ath9k 的 active monitor 基础能力可用。
- `tcpdump` radiotap 抓包可用。
- OWL 在 channel 6 上可以通过手动 active monitor + `-N` 路径创建 `awdl0`。
- OWL 自己的 AWDL action frame 能在本机 monitor 抓包中看到，说明发送路径至少进入了无线接口。
- 当时尚未证明 iPhone 和 OWL 互相发现，也尚未证明 `awdl0` 上出现来自 Apple peer 的 IPv6/mDNS 流量。
- 当时的下一步重点不是 UxPlay，而是先让 iPhone peer 出现在 OWL 日志或 `ip -6 neigh show dev awdl0` 里。

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
- 当前最可能的下一步不是 Avahi/UxPlay，而是继续确认 iPhone 是否真的进入 AWDL 发现状态，以及当前 AR928X/ath9k 是否能在实际 Apple AWDL 场景里看到对端。
- 下次重测时建议明确使用 iPhone 的 AirDrop/隔空投送触发：打开照片或文件的分享面板，进入 AirDrop 页面，接收设置临时设为“所有人/Everyone for 10 Minutes”，让手机靠近树莓派并保持屏幕点亮。
- 如果这样仍然在 6/44/149 都看不到外部 vendor action frame，建议引入第二块已知能抓 Apple AWDL 的监听设备或等待新购 AR9280 到货后复测，用来区分 iPhone 触发问题和当前 AR928X/ath9k/转接板组合问题。

### 2026-04-29 两个关键假设

当前有两个主要假设，二者都合理，不能只靠现有结果排除其中一个。

假设 A：当前 AR928X 看起来具备能力，但实际不适合本机 OWL 场景。

支持这个假设的现象：

- `iw phy` 显示 active monitor 能力，手动切换也成功，但 OWL 自动切换 monitor 时会遇到 `Object busy`，只能走 `-N` workaround。
- channel 44/149 当前被标记为 `No IR`，不能作为主动发射信道使用。
- channel 6 能发送 OWL 自己的 vendor action frame，但没有发现 iPhone peer。
- OWL README 推荐/测试过的是 AR9280；当前实测卡按你的口径是 AR928X，所以待到货 AR9280 有明确的 A/B 对比价值。

验证这个假设的最好办法：

- 等新网卡到货后，在同一台 Pi、同一系统、同一 iPhone、同一位置重复完全相同的 6/44/149 抓包和 OWL peer 测试。
- 如果新购 AR9280 能看到 iPhone AWDL vendor action frame 或能 add peer，而当前 AR928X 不能，则问题基本落在当前 AR928X/ath9k/转接板组合。
- 如果两张网卡都看不到 iPhone AWDL vendor action frame，则硬件单点嫌疑下降，转向 iPhone 触发方式、iOS 版本或 OWL 协议兼容性。

假设 B：iOS 26 上 AWDL/AirDrop/AirPlay 相关行为已经和当前 OWL 不兼容，或者至少更难触发。

支持这个假设的理由：

- OWL 是逆向实现，README 已明确提示它可能不兼容未来 AWDL 版本。
- OpenDrop/OWL 生态也明确存在发现层限制：Linux 侧不能像 Apple 设备一样通过 BLE 广播完整触发对端 AWDL。
- Apple 官方 AirDrop 行为在新系统里有更多限制，例如“所有人 10 分钟”和面向非联系人的 AirDrop code，这些不一定改变 AWDL 帧格式，但会影响发现/握手路径。
- OWL 的代码和论文时代主要对应较早的 iOS/macOS AWDL 行为；没有证据说明它已经针对 iOS 26 做过持续兼容。

验证这个假设的最好办法：

- 找一台较旧 iOS/iPadOS 设备，或一台 macOS 设备，重复同样的 AirDrop/AWDL 抓包。
- 用两台 Apple 设备互相 AirDrop，同时用 `wlan1` 纯监听 6/44/149。目标不是让 OWL 参与，而是确认当前网卡能否被动看到真实 Apple AWDL 帧。
- 如果 Apple-to-Apple AirDrop 发生时，当前 AR928X 仍然看不到任何 vendor-specific action frame，那么优先怀疑监听硬件/信道/抓包方式。
- 如果能看到 Apple-to-Apple AWDL 帧，但 OWL 加不进 peer，则问题更偏向 OWL 协议兼容性或 active monitor/injection/ACK。
- 如果旧系统能和 OWL add peer，而 iOS 26 不行，则 iOS 26 兼容性假设成立。

建议的下一轮最小验证矩阵：

| 测试 | 设备/网卡 | 预期能回答的问题 |
| --- | --- | --- |
| Apple-to-Apple AirDrop 旁路抓包 | 当前 AR928X 纯 monitor | 当前卡能不能看到真实 Apple AWDL 帧 |
| OWL vs 旧 iOS/macOS | 当前 AR928X | OWL 是否仍能和较旧 Apple AWDL 实现通信 |
| OWL vs iOS 26 | 新购 AR9280 | 当前失败是否是 AR928X 单点问题 |
| Apple-to-Apple AirDrop 旁路抓包 | 新购 AR9280 | 新卡是否比当前 AR928X 更适合 AWDL 抓包 |

判断标准：

- 如果“Apple-to-Apple AirDrop 旁路抓包”都看不到 AWDL 帧，不要继续调 UxPlay。
- 如果能旁路看到 Apple AWDL 帧，但 OWL 没有 peer，继续看 OWL 协议兼容、channel sequence、ACK/injection。
- 如果 OWL 能和旧 iOS/macOS 通，但不能和 iOS 26 通，才进入 iOS 26 协议差异分析。
- 如果 OWL 能和 iOS 26 add peer，再继续推进 Avahi/UxPlay over `awdl0`。

### 2026-04-29 重新验证结果

本轮测试前，iPhone 侧已忽略所有 Wi-Fi 网络，只保持 Wi-Fi 和 Bluetooth 开启。测试目标是重新验证“当前 AR928X/ath9k + Pi 5 + OWL 是否能看到 Apple AWDL peer，并把它映射到 `awdl0`”。

重要修正：直接使用 `wlan1` 作为 OWL 接口不可靠。本轮发现 `iw dev wlan1 info` 显示频道已经切到 6/11/149，但 `tcpdump` 的 radiotap 频率仍然持续显示 `2412 MHz`，实际抓到的是 channel 1 的 beacon/probe response。这会让之前“固定 channel 6/44/149 纯监听但看不到 AWDL”的结果带有误导性。

更可靠的本机路径是：

```sh
sudo nmcli device set wlan1 managed no
sudo nmcli device set p2p-dev-wlan1 managed no

sudo ip link set wlan1 down
sudo iw dev wlan1 interface add monawdl type monitor flags active
sudo ip link set monawdl up
sudo iw dev monawdl set channel 6 HT20

sudo ./build/daemon/owl -i monawdl -h awdl0 -c 6 -vv -f -N
```

关键结果：

- 只保留 `monawdl` 处于 UP、`wlan1` down 后，radiotap 频率能正确跟随设置。例如设置 channel 149 后，`tcpdump -i monawdl` 显示 `5745 MHz`。
- 用 `monawdl` 启动 OWL 后，OWL 立即收到外部 AWDL PSF。
- 日志中出现多个 peer，例如 `32:21:f2:ae:d5:39`、`46:23:d6:59:0b:dc`、`c6:96:0f:47:c4:f1`。
- `ip -6 neigh show dev awdl0` 出现永久邻居项，例如 `fe80::3021:f2ff:feae:d539 lladdr 32:21:f2:ae:d5:39 PERMANENT`。
- `tcpdump -i monawdl` 能抓到外部 AWDL vendor action frame，频率为 `2437 MHz`。
- `ping -6 -I awdl0 fe80::3021:f2ff:feae:d539` 成功，3 发 3 收，0% packet loss。
- Avahi daemon 正在运行，`avahi-browse -a -t` 能看到 `awdl0 IPv6 OpenClaw02 ... Workstation local`。
- 随后已经进入 Avahi/UxPlay 阶段；当前安装的 UxPlay 版本为 Debian trixie 源里的 `1.71.1-1`。

本轮结论：

- “当前 AR928X/ath9k 完全不能用于 OWL”这个假设被明显削弱。至少在 `monawdl` 路径下，AWDL peer 发现、IPv6 neighbor 添加、link-local ping 都已经跑通。
- “iOS 26 完全不能和 OWL 通信”这个假设也被削弱。当前已经能收到 Apple AWDL PSF，并能 ping 通其中一个 AWDL peer。
- 真正暴露出来的新问题是：不能直接相信 `iw dev wlan1 info` 的频道显示，必须用 radiotap 频率做二次确认。
- 下一步应该进入 Avahi/UxPlay 阶段：让 AirPlay 接收端服务明确发布到 `awdl0`，并观察 iPhone 的 Screen Mirroring 是否通过 `awdl0` 发送 mDNS 和 RTSP 连接。
- 仍然需要保留待到货 AR9280 的 A/B 测试价值，因为它可以验证 `monawdl` workaround 是否是当前卡/驱动/接口状态特有的问题。

### 2026-04-29 Avahi/UxPlay 阶段验证结果

本轮目标是先验证 Linux 侧能否把 AirPlay 接收端发布到 `awdl0`，再观察 iPhone 是否会通过 AWDL 进入发现和连接流程。

已完成项：

- 已安装 `uxplay 1.71.1-1`，`avahi-daemon` 处于 active。
- 使用 `monawdl` 路径启动 OWL 后，`awdl0` 获得 IPv6 link-local 地址：`fe80::b674:9fff:fe6c:bc2f/64`。
- UxPlay 以测试渲染方式启动成功：

```sh
uxplay -n OwlPi-AWDL -nh -p 41000 -vs fakesink -as fakesink -nohold -d
```

说明：在 Codex 普通 sandbox 中直接运行 UxPlay 会报 `Error initialising socket 1` / `dnssd_register_raop failed with error code -65553`，但在完整系统权限下运行成功。这更像是 sandbox 权限/网络命名空间问题，不是 UxPlay 参数本身失败。

Avahi/DNS-SD 发布结果：

- `avahi-browse -a -t -r` 能看到 UxPlay 服务出现在 `awdl0 IPv6` 上。
- `_airplay._tcp` 服务名为 `OwlPi-AWDL`，地址为 `fe80::b674:9fff:fe6c:bc2f`，端口为 `41001`。
- `_raop._tcp` 服务名为 `2CCF67A0257B@OwlPi-AWDL`，地址同样为 `fe80::b674:9fff:fe6c:bc2f`，端口为 `41001`。
- TXT record 中能看到 `model=AppleTV3,2`、`features=0x527FFEE6,0x0`、`deviceid=2c:cf:67:a0:25:7b`、`srcvers=220.68` 等字段。
- `tcpdump -i awdl0` 能看到本机 Avahi 在 `awdl0` 上发送 `_airplay._tcp.local` 和 `_raop._tcp.local` 的 mDNS 查询/响应。

本轮通过结论：

- Linux 侧“UxPlay + Avahi 发布到 `awdl0 IPv6`”已经验证通过。
- 这说明只要 OWL 提供 `awdl0`，普通 DNS-SD/AirPlay 接收端栈确实可以把 AWDL 当成一张本地 IPv6 网卡来发布服务。

本轮未通过/未完成项：

- 90 秒抓包期间，`ip -6 neigh show dev awdl0` 没有出现新的 iPhone peer。
- `tcpdump -i awdl0 'ip6 and (udp port 5353 or tcp port 41001 or icmp6)'` 只看到本机 Avahi/mDNS 流量，没有看到 iPhone 发来的 mDNS query 或 TCP 连接。
- `tcpdump -i monawdl` 针对外部 AWDL vendor action frame 的过滤结果为 0 包。
- 因此还不能证明 iPhone 已经通过 AWDL 发现或连接了 `OwlPi-AWDL`。

下一轮 Avahi/UxPlay 验证重点：

1. 重新启动 OWL + UxPlay 后，先要求 `ip -6 neigh show dev awdl0` 出现 Apple peer。
2. iPhone 侧打开 Screen Mirroring，并可先打开 AirDrop/分享面板来强制触发 AWDL。
3. 如果 iPhone 列表里出现 `OwlPi-AWDL`，必须同时抓到 `awdl0` 上来自 iPhone 的 mDNS 或 TCP 41001 流量，才能判定发现路径真的走 AWDL。
4. 如果列表不出现，而 OWL 已经有 peer，则重点查 Avahi 是否响应了来自 peer 的 `_airplay._tcp` / `_raop._tcp` 查询。
5. 如果列表出现但连接失败，则进入 UxPlay IPv6 监听、端口、防火墙、RTSP/pairing 日志排查。

### 2026-04-29 UxPlay LAN 基线与 AWDL 触发复测

用户已手动确认：在普通同 Wi-Fi 网络下，UxPlay 可以被 iPhone 发现，并且可以正常投屏显示。因此 UxPlay 基础功能和 iOS 26 的普通 AirPlay 投屏路径已经通过，不再是当前主嫌疑点。

本轮继续验证“Wi-Fi 未连接热点，只开启 Wi-Fi/Bluetooth 时，iPhone 是否会通过 Screen Mirroring 或 AirDrop 触发 AWDL”。

关键观察：

- 用户此前一直停留在 Screen Mirroring 界面，iPhone Wi-Fi 未连接具体热点，Bluetooth 开启；树莓派侧没有看到来自 iPhone 的 `awdl0` neighbor、mDNS query 或 TCP 连接。
- 本轮准备纯监听时，发现当前 AR928X/ath9k 出现频道控制异常：`iw dev wlan1 info` 或 `iw dev monawdl info` 显示已经切到 channel 6/11/149，但 `tcpdump` radiotap 仍持续显示 `2412 MHz`，也就是 channel 1。
- 即使只保留 `wlan1` 自身为 monitor，不创建 `monawdl`，`iwconfig wlan1` 显示 `Frequency:5.745 GHz` 时，`tcpdump -i wlan1` 仍然抓到 `2412 MHz` 的 beacon/probe/data。
- 因此当前会话里不能把无线侧 `tcpdump` 当作可靠的 channel 6/44/149 旁路观察证据。

随后改用 OWL 自身作为判据：

```sh
sudo iw dev wlan1 set channel 6 HT20
timeout 120 sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 6 -vv -f -N
```

测试期间要求 iPhone 进入 AirDrop 分享页面，并尽量触发附近设备发现。

结果：

- OWL 能创建 `awdl0`，并持续发送 MIF/PSF。
- `awdl0` 有本机 IPv6 link-local 地址。
- `awdl0` 上只看到本机发出的 IPv6/mDNS/ICMPv6 流量。
- `ip -6 neigh show dev awdl0` 一直为空。
- OWL 日志没有出现 `add peer ...`，也没有看到可用外部 peer。
- 120 秒后测试超时结束，`awdl0` 被清理，`wlan1` 已恢复为 managed/disconnected。

本轮结论：

- “UxPlay 本身不能被 iOS 26 使用”已排除。
- 当前阻塞点回到 AWDL 层：iPhone 没有被当前 OWL/AR928X 组合稳定发现，或者当前 AR928X/ath9k 的实际频道/监听/注入状态不可靠。
- 在这个状态下继续调 Avahi/UxPlay 收益不大，因为 AirPlay 发现必须先建立在 `awdl0` 能看到 Apple peer 的基础上。

下一步建议：

1. 优先等待新购 AR9280 到货，按同样步骤做 A/B 对比。
2. 如果要继续用当前 AR928X 排查，先做“Apple-to-Apple AirDrop 旁路抓包”：用两台 Apple 设备互传 AirDrop，同时用当前卡监听，目标是确认它到底能不能看到真实 Apple AWDL 帧。
3. 如果旁路抓包仍然只显示 channel 1 或看不到 AWDL，则重点怀疑当前 AR928X/驱动/转接板状态，而不是 AirPlay 或 UxPlay。
4. 如果旁路抓包能看到 Apple AWDL，但 OWL 仍不能 add peer，再继续看 OWL 兼容性、ACK/injection 和 iOS 26 差异。

### 2026-04-29 监管域与 `no IR` 复测

本轮目标是验证能否通过正常监管域配置解除 channel 44/149 的 `no IR` 限制。

执行前状态：

```text
global country CN
phy#1 country 99: DFS-UNSET
phy#1 5GHz: PASSIVE-SCAN
channel 44: no IR
channel 149: no IR
```

已尝试的合法配置：

```sh
sudo iw reg set CN
iw reg get
iw phy phy1 info
```

结果：

- `global` 仍为 `country CN`。
- `phy#1` 仍为 `country 99: DFS-UNSET`。
- 5GHz 仍显示 `PASSIVE-SCAN`。
- channel 44 仍显示 `no IR`。
- channel 149 仍显示 `no IR`。

随后对 `wlan1` 做了温和的 down/up 刷新：

```sh
sudo ip link set wlan1 down
sudo ip link set wlan1 up
```

再次复查后结果仍然不变。

本轮阶段性结论：

- 仅通过 `sudo iw reg set CN` 不能解除当前 AR928X/ath9k 的 44/149 `no IR` 限制。
- 当时当前卡的 `phy#1` 没有跟随 global `CN` 监管域切换，仍保持 `country 99` 和 5GHz passive-scan 约束。
- 后续继续验证发现：在 global `CN` 已存在的前提下，重载 ath9k 模块可以让 44/149 的 `no IR` 消失。见下一节。

### 2026-04-29 ath9k 重载后解除 `no IR`

本轮目标是在不修改驱动、EEPROM 或 wireless-regdb 的前提下，尝试让 ath9k 在 `CN` 监管域已存在的状态下重新初始化。

先确认已有条件：

- `/proc/cmdline` 已包含 `cfg80211.ieee80211_regdom=CN`。
- `wireless-regdb` 已安装，版本为 `2026.02.04-1~deb13u1`。
- `/lib/firmware/regulatory.db` 指向 Debian regulatory.db。
- 启动日志中 ath9k 读到 `EEPROM regdomain: 0x6b`，并出现 `ath_regd_init` warning；这解释了为什么简单 `iw reg set CN` 不足以让 `phy` 立即跟随。

执行的非破坏性重载流程：

```sh
nmcli device set wlan1 managed no
nmcli device set p2p-dev-wlan1 managed no
sudo ip link set wlan1 down

sudo modprobe -r ath9k
sudo modprobe -r ath9k_common
sudo modprobe -r ath9k_hw
sudo modprobe -r ath

sudo iw reg set CN
sudo modprobe ath9k
```

重载后结果：

- 外接卡重新枚举为 `phy#2`，接口仍为 `wlan1`。
- `iw phy phy2 info` 中 channel 44 不再显示 `no IR`。
- `iw phy phy2 info` 中 channel 149 不再显示 `no IR`。
- channel 149 功率显示为 `18.0 dBm`，低于之前的 `33.0 dBm` 标称值，但已不再是 passive/no-IR 状态。

随后用 OWL 做短启动验证：

```sh
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo iw dev wlan1 set monitor active
sudo ip link set wlan1 up
sudo iw dev wlan1 set channel 149 HT20
timeout 5 sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 149 -vv -f -N
```

OWL 输出关键结果：

```text
Channel 149 [5745 MHz] is available for frame injection
```

本轮结论：

- 当前会话内，ath9k 模块重载后，44/149 的 `no IR` 已经解除。
- OWL 已确认 channel 149 可以用于 frame injection。
- 这条路径没有修改驱动、EEPROM 或 regulatory.db，但它可能不是永久状态；重启后仍需复查 `iw phy ... info`。
- 如果重启后恢复为 `no IR`，可以把“设置 `CN` 后重载 ath9k”作为进入 AWDL 验证前的准备步骤。

### 2026-04-29 44/149 可发射后的 UxPlay 发现复测

本轮目标是在 channel 44 和 149 都已经不再显示 `no IR` 的前提下，重新跑 OWL + UxPlay，看 iPhone 是否能在非同 Wi-Fi 场景通过 AWDL 发现 `OwlPi-AWDL`。

启动前状态：

- `iw phy phy2 info` 中 channel 44 显示为 `5220.0 MHz [44] (20.0 dBm)`，不再带 `no IR`。
- `iw phy phy2 info` 中 channel 149 显示为 `5745.0 MHz [149] (18.0 dBm)`，不再带 `no IR`。
- `wlan1` 是外接 AR928X/ath9k，MAC 为 `b4:74:9f:6c:bc:2f`。
- `wlan0` 仍连接普通局域网，用于远程操作；AWDL 测试使用 `wlan1`。

channel 149 验证：

```sh
nmcli device set wlan1 managed no
nmcli device set p2p-dev-wlan1 managed no
sudo ip link set wlan1 down
sudo iw dev wlan1 set type monitor
sudo iw dev wlan1 set monitor active
sudo ip link set wlan1 up
sudo iw dev wlan1 set channel 149 HT20
sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 149 -vv -f -N
```

关键结果：

- OWL 输出 `Channel 149 [5745 MHz] is available for frame injection`。
- `awdl0` 创建成功，IPv6 link-local 地址为 `fe80::b674:9fff:fe6c:bc2f/64`。
- UxPlay 启动成功，监听 `0.0.0.0:41001` 和 `[::]:41001`。
- `avahi-browse -a -t -r` 能看到：
  - `awdl0 IPv6 OwlPi-AWDL _airplay._tcp`，地址 `fe80::b674:9fff:fe6c:bc2f`，端口 `41001`。
  - `awdl0 IPv6 2CCF67A0257B@OwlPi-AWDL _raop._tcp`，地址同上，端口 `41001`。
- `tcpdump -i awdl0 'ip6 and (udp port 5353 or tcp port 41001 or icmp6)'` 抓包 120 秒，只看到本机发出的 mDNS/Router Solicitation，没有看到 iPhone 入站 mDNS、NDP 或 TCP 41001。
- `ip -6 neigh show dev awdl0` 为空。

channel 44 验证：

```sh
sudo iw dev wlan1 set channel 44 HT20
sudo ./build/daemon/owl -i wlan1 -h awdl0 -c 44 -vv -f -N
```

关键结果：

- OWL 输出 `Channel 44 [5220 MHz] is available for frame injection`。
- `awdl0` 再次创建成功，仍是 `fe80::b674:9fff:fe6c:bc2f/64`。
- Avahi/UxPlay 再次发布到 `awdl0 IPv6`，端口仍为 `41001`。
- `tcpdump -i awdl0` 抓包 120 秒，只看到本机发出的 mDNS/Router Solicitation，没有看到 iPhone 入站 mDNS、NDP 或 TCP 41001。
- `ip -6 neigh show dev awdl0` 为空。

本轮需要注意的干扰项：

- 复测过程中，`avahi-browse` 一度在 `wlan0` 上看到 `ouhyoutekiiPhone.local / 192.168.0.105`。如果这就是测试 iPhone，说明手机当时仍连接普通 Wi-Fi；这种情况下 Screen Mirroring 里看到 `OwlPi-AWDL` 不能证明 AWDL 成功，因为它可能走的是普通 LAN。
- 纯 AWDL 验证时，需要确保 iPhone 不连接任何热点，同时用 `avahi-browse` 确认 `wlan0` 上没有该 iPhone 的服务记录。

本轮结论：

- 44/149 的 `no IR` 限制已经不是当前直接阻塞点；OWL 能在两个频道上启动并确认可以 frame injection。
- Avahi/UxPlay 发布到 `awdl0 IPv6` 已再次验证通过。
- 失败点仍然在 AWDL peer/link-local 建立阶段：本轮没有看到 iPhone 作为 `awdl0` IPv6 neighbor，也没有看到 iPhone 通过 `awdl0` 发来的 mDNS 或 AirPlay TCP 连接。
- 下一步不应继续优先调 UxPlay 参数，而应回到 AWDL 触发和 peer 建立：确认 iPhone 是否真的进入 Apple AWDL 发现状态，或用 Apple-to-Apple AirDrop/Screen Mirroring 做旁路抓包；如果新购对比 AR9280 到货，也应优先复测是否能稳定看到 peer。

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

当前文档不再记录早先可能误记的购买型号，只按实际验证对象区分：当前测试卡记为 AR928X，新购到货后的卡作为对比卡单独记录。

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
- 2026-04-29 重新验证后，更可靠的 workaround 是创建独立 active monitor 接口 `monawdl`，把原 `wlan1` down 掉，再用 `owl -i monawdl ... -N` 启动。
- 直接使用 `wlan1` 时，`iw dev wlan1 info` 显示的频道可能和 `tcpdump` radiotap 里的实际频率不一致；必须用 radiotap 频率确认真实监听/发射频道。
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
sudo nmcli dev set p2p-dev-wlan1 managed no
sudo rfkill unblock wifi
```

通过标准：

- 没有 wpa_supplicant/NetworkManager 进程立刻重新配置 `wlan1`。
- 如果需要普通联网，使用以太网或另一块 Wi-Fi。
- 用 `tcpdump` 的 radiotap 频率确认实际频道没有被后台扫描拉走。

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
sudo nmcli dev set p2p-dev-wlan1 managed no
sudo ip link set wlan1 down
sudo iw dev wlan1 interface add monawdl type monitor flags active
sudo ip link set monawdl up
sudo iw dev monawdl set channel 6 HT20
sudo ./build/daemon/owl -i monawdl -h awdl0 -c 6 -vv -f -N
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
- `iw dev ... info` 显示在 channel 6/149，但 `tcpdump` radiotap 仍显示 `2412 MHz`，说明实际频道控制不可信；优先改用独立 `monawdl` 并把 `wlan1` down。

### 5. 验证监管域和频道选择

先从 channel 6 开始，因为它能避开很多 5 GHz 监管域和驱动问题：

```sh
iw reg get
iw list
sudo ./build/daemon/owl -i monawdl -h awdl0 -c 6 -vv -f -N
```

channel 6 跑通后，再重复测试 44 或 149。

通过标准：

- 选定频道没有被标记为 `disabled` 或 `no IR`。
- OWL 没有警告该频道不能注入。
- 换频道后仍能发现 Apple peer。
- `tcpdump -i monawdl` 显示的 radiotap 频率与目标频道一致，例如 channel 6 应为 `2437 MHz`。

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

当前 `monawdl` 路径下已经实测通过：

```text
fe80::3021:f2ff:feae:d539 lladdr 32:21:f2:ae:d5:39 PERMANENT
ping -6 -I awdl0 fe80::3021:f2ff:feae:d539 -> 3 transmitted, 3 received
```

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

当前状态：

- `uxplay 1.71.1-1` 已安装。
- `avahi-daemon` active。
- 本轮重点先验证 AWDL 侧发布，所以普通 LAN 下真实画面渲染仍建议单独补一轮；当前用的是 `fakesink`，适合验证发现/监听，不验证显示输出。

这个基线没跑通前，不要把 AirPlay 协议问题和 AWDL 链路问题混在一起排查。

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

本机当前更适合先用固定名称和测试 sink 做发现验证：

```sh
uxplay -n OwlPi-AWDL -nh -p 41000 -vs fakesink -as fakesink -nohold -d
```

通过标准：

- `avahi-browse` 显示 UxPlay 服务发布在 `awdl0` 上，并且有 IPv6 地址。
- `awdl0` 上能看到 mDNS query 和 response。
- Avahi 响应时，OWL 的 multicast TX 计数或日志有增长。

当前状态：

- 已确认 `avahi-browse -a -t -r` 能看到 `_airplay._tcp` 和 `_raop._tcp` 发布在 `awdl0 IPv6` 上。
- 已确认 `tcpdump -i awdl0` 能看到本机 Avahi 对 `_airplay._tcp.local` / `_raop._tcp.local` 的 mDNS 查询/响应。
- 尚未看到 iPhone 通过 `awdl0` 发来的 mDNS query 或 TCP 41001 连接。

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
- 确认 OWL 实际使用的是独立 monitor 接口 `monawdl`，并且原 `wlan1` 已 down。
- 用无线侧 `tcpdump` 的 radiotap 频率确认当前确实在 channel 6 的 `2437 MHz`，不要只看 `iw dev ... info`。

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

截至 2026-04-29，使用 `monawdl` 路径时，上面这些底层条件曾经通过一次。ath9k 重载后，channel 44/149 的 `no IR` 限制也已在当前会话解除，并且 OWL 能在 44/149 上确认 frame injection 可用。Avahi/UxPlay 阶段已经多轮验证：服务以 IPv6 形式发布到 `awdl0` 已通过；iPhone 是否会通过 `awdl0` 访问 AirPlay 接收端仍未通过。

如果这些通过但 UxPlay 不能被发现，重点看 Avahi/UxPlay 的 IPv6 DNS-SD 接口绑定。

如果 UxPlay 能被发现但连不上，重点看 UxPlay IPv6 监听、防火墙端口和 AirPlay 协议兼容性。

如果接收端路径强依赖 IPv4，这个不改代码的方案就不够。当前 OWL 发送路径把 payload 标成 IPv6，所以 IPv4 over AWDL 需要改 OWL 代码，或者换一条接收/发现策略。

## 参考资料

- 本仓库 OWL README：`README.md`
- UxPlay upstream README: https://github.com/FDH2/UxPlay
- OWL upstream README: https://github.com/seemoo-lab/owl
- OpenDrop limitations and compatibility notes: https://deepwiki.com/seemoo-lab/opendrop/1.3-limitations-and-compatibility
- Apple Support: Use AirDrop on your iPhone or iPad: https://support.apple.com/en-us/ht204144
- Linux cfg80211 monitor flags documentation: https://docs.kernel.org/driver-api/80211/cfg80211.html
- Wi-Fi injection/active monitor testing notes: https://github.com/vanhoefm/wifi-injection
- Raspberry Pi forum thread on Atheros PCIe DMA failure: https://forums.raspberrypi.com/viewtopic.php?t=343846
- Launchpad bug about `pcie-32bit-dma` workaround: https://bugs.launchpad.net/bugs/1927037
