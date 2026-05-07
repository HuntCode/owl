# AirPlay over OWL/AWDL Validation

Date: 2026-04-28
Last updated: 2026-05-07

## 目标和边界

目标是在 Raspberry Pi 5 上用 OWL 提供 AWDL 链路，让 UxPlay 这类 AirPlay 接收端在 iPhone 不加入同一个 Wi-Fi 热点时也能被发现和投屏。

```text
普通同 Wi-Fi:
iPhone -> mDNS/DNS-SD -> _airplay._tcp/_raop._tcp -> TCP/RTSP/RTP -> UxPlay

目标 AWDL:
iPhone -> AWDL peer/link-local IPv6 -> awdl0 -> mDNS/DNS-SD -> UxPlay
```

分工：

- OWL 负责 AWDL 和 `awdl0`，不实现 AirPlay。
- UxPlay 负责 AirPlay 协议和渲染，不实现 AWDL。
- Avahi 负责在 `awdl0` 上发布和响应 Bonjour/mDNS。
- 当前按 IPv6 link-local 验证，不预期 AWDL 上出现 IPv4/DHCP 网段。

## 当前进展

### 已确认可用

- 普通同 Wi-Fi LAN 下，iPhone 可以发现 UxPlay 并正常投屏，说明 UxPlay 基础 AirPlay 能力可用。
- 当前外接卡按文档记为 AR928X，PCI ID `Qualcomm Atheros AR928X [168c:002a]`，驱动 `ath9k`，接口 `wlan1`。
- Raspberry Pi 5 加入 `dtoverlay=pcie-32bit-dma-pi5` 后，之前的 `ath9k ... Failed to allocate tx descriptors: -12` 已解决。
- `ath9k` 支持 monitor、active monitor、`set_channel`、`frame`，具备 OWL 所需的基础能力。
- channel 149 的 `no IR` 已通过完整 ath9k 模块重载在当前会话解除，OWL 确认 `Channel 149 [5745 MHz] is available for frame injection`。
- `monawdl` radiotap 已确认真实频率为 `5745 MHz`，不是被驱动拉回到 2.4 GHz channel 1。
- channel 149 上 OWL 能发现多个 AWDL peer，`awdl0` 能产生 IPv6 neighbor，并能 ping 通部分 link-local peer。
- Avahi/UxPlay 能发布到 `awdl0 IPv6`，并且 iPhone 的 Screen Mirroring 已能看到 `OwlPi-P2P149-HT20`。
- OWL 的 Linux `set_channel()` 已从 `HT40PLUS` 改为 `HT20`；否则 OWL 启动后会把手动设置的 `149 HT20` 改成 `149 HT40`，iPhone 发现会变得不稳定。

### 当前阻塞点

现象：iPhone 能发现接收端，但点击后立即弹出无法连接。

测试中已看到：

- `_airplay._tcp`、`_raop._tcp` 能发布到 `awdl0 IPv6`。
- 手动补充 `_airplay-p2p._tcp`，且服务名改成常见的 `DeviceID@Name` 形式后，iPhone 仍能发现设备；但 `_airplay-p2p._tcp` 不是可见性的必要条件。
- 单独 `_airplay._tcp` 不会出现在 iPhone 屏幕镜像列表中；`_raop._tcp` 仍是发现候选的重要输入。
- 使用 Mac-like `_airplay/_raop` TXT 后，点击时 `awdl0` 能看到 iPhone 连接 TCP 7000：iPhone 发 SYN，Pi 回 SYN-ACK，随后 iPhone 主动 RST，并且该实例会从 iPhone 列表临时消失。
- UxPlay/AppleTV3,2 风格 TXT 在 AWDL-only 场景下反而可能不显示，说明 iOS 26.4 对 AWDL AirPlay 候选有额外筛选。

当前判断：

```text
149/no-IR、真实频道、AWDL peer、IPv6 link-local、mDNS 发现
都已可进入可复现验证阶段。

新的阻塞点在 iOS 点击后的 AirPlay P2P 资格/握手阶段：
部分记录组合能让 iPhone 发起 TCP，但 iPhone 会在 SYN-ACK 后主动 RST。
```

为了保持发现列表稳定，提交基线先使用 `OwlPi-P2P149-HT20` 的 UxPlay 原生 `_airplay/_raop` 发布，再手动补 `_airplay-p2p._tcp`；Mac-like / hybrid TXT 组合只作为实验，不作为主线复现流程。

## 关键操作

### 重启后最小复现流程

重启 Raspberry Pi 或重新插拔外接网卡后，按下面顺序从零恢复：

```sh
# 1. 解除 NetworkManager 对外接卡的接管
nmcli device set wlan1 managed no
nmcli device set p2p-dev-wlan1 managed no
sudo ip link set wlan1 down

# 2. 重载 ath9k，让 channel 149 去掉 no IR
sudo modprobe -r ath9k
sudo modprobe -r ath9k_common
sudo modprobe -r ath9k_hw
sudo modprobe -r ath
sudo iw reg set CN
sudo modprobe ath9k

# 3. 找到外接卡新的 phy 编号，并确认 149 无 no IR
iw dev
iw phy phyX info

# 4. 创建普通 monitor，固定 149 HT20
sudo ip link set wlan1 down
sudo iw dev wlan1 interface add monawdl type monitor
sudo ip link set monawdl up
sudo iw dev monawdl set channel 149 HT20

# 5. 启动 OWL。必须用 -N 保持外部创建的普通 monitor。
sudo ./build/daemon/owl -i monawdl -h awdl0 -c 149 -v -f -N
```

AirPlay 发现测试另开终端：

```sh
uxplay -n OwlPi-P2P149-HT20 -nh -m b4:74:9f:6c:bc:2f -p -d -vs 0 -as 0

sudo avahi-publish-service -s 'B4749F6CBC2F@OwlPi-P2P149-HT20' \
  _airplay-p2p._tcp 7000 \
  'deviceid=b4:74:9f:6c:bc:2f' \
  'features=0x4A7FCFD5,0x38174FDE' \
  'flags=0x20004' \
  'model=MacBookPro15,1' \
  'srcvers=870.14.1' \
  'pk=ea765eb6d570b6b94e4db5c61b4d92ca0e89dd4ae9d598bc3ba55261f91bfedf' \
  'vv=2'
```

说明：

- 这个组合用于“发现层稳定复现”：即使点击后仍无法投屏，也应优先保证 iPhone 列表能持续看到 `OwlPi-P2P149-HT20`，便于继续验证通道、mDNS 和 AWDL peer 状态。
- 不要把 Mac-like `_airplay/_raop` 手工组合当作提交基线。该路径能推进到 TCP SYN/SYN-ACK/RST，但失败后 iPhone 会临时隐藏该服务实例，影响重复验证。
- 如果列表不显示，先换一个新服务名并确认 `avahi-browse` 只在 `awdl0 IPv6` 上看到目标记录，避免 iPhone 本地失败缓存干扰。

验证点：

```sh
iw dev monawdl info
iw dev monawdl survey dump
sudo tcpdump -i monawdl -e -n -vv 'wlan[0] == 0xd0'
sudo avahi-browse -a -t -r
sudo tcpdump -i awdl0 -n -vv 'ip6 and (udp port 5353 or tcp port 7000 or tcp port 7001 or tcp port 7100 or icmp6)'
```

通过标准：

- `iw dev monawdl info` 为 `channel 149 (5745 MHz), width: 20 MHz, center1: 5745 MHz`。
- `survey dump` 为 `5745 MHz [in use]`。
- radiotap 输出包含 `5745 MHz`。
- `avahi-browse` 在 `awdl0 IPv6` 上能看到 `_airplay._tcp`、`_raop._tcp`、`_airplay-p2p._tcp`。
- iPhone 屏幕镜像列表能看到 `OwlPi-P2P149-HT20`。

### 解除 channel 149 的 no IR

问题：channel 149 如果显示 `no IR`，Linux 监管域不允许主动发射，OWL 会提示不能 frame injection。AWDL 只能“听见”不够，点击后的会话需要能主动发帧。

仅执行 `sudo iw reg set CN` 不足以让当前 phy 立即解除 `no IR`。本机有效流程是完整卸载并重载 ath9k 相关模块：

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
iw dev
iw phy phyX info
```

通过标准：

- 外接卡重新出现，例如 `phy#3 / wlan1`。
- `iw phy phyX info` 中 channel 149 不再显示 `no IR`。
- OWL 启动时确认 `Channel 149 [5745 MHz] is available for frame injection`。

注意：

- 只重载 `ath9k` 不够。
- `iw reg get` 仍可能显示该 phy 为 `country 99 ... PASSIVE-SCAN`；实际可用性以 `iw phy phyX info` 的 per-channel 标记和 OWL 的 frame-injection 检查为准。
- 这个状态不保证重启后保留，重启或重新插卡后必须复查。

### 用 radiotap 确认真实频道

问题：`iw dev ... info` 显示的频道不一定等于真实发射/监听频道。之前出现过“设置 36，但 radiotap 看到 2412 MHz/channel 1”的现象，说明频道可能被内核或驱动拉回去。

因此每次启动 OWL 前都要确认无线侧真实频率：

```sh
nmcli device set wlan1 managed no
nmcli device set p2p-dev-wlan1 managed no
sudo ip link set wlan1 down

sudo iw dev wlan1 interface add monawdl type monitor
sudo ip link set monawdl up
sudo iw dev monawdl set channel 149 HT20
iw dev monawdl info
iw dev monawdl survey dump
sudo tcpdump -i monawdl -e -n -vv 'wlan[0] == 0xd0'
```

通过标准：

- `iw dev monawdl info` 显示 channel 149。
- `iw dev monawdl survey dump` 显示 `5745 MHz [in use]`。
- `tcpdump` radiotap 中出现 `5745 MHz`。
- OWL 日志确认 channel 149 可 frame injection。

作用：

- 排除“看起来设置了 149，实际还在 1/6/36”的误判。
- 确认 iPhone 的 AWDL channel sequence 和 OWL 的实际工作频道能对上。
- 避免把底层频道错误误判成 UxPlay 或 AirPlay 协议错误。

关键发现：

- 不要用 `flags active` 创建 `monawdl`。实测 `iw dev` 仍会显示 149，但 radiotap 和 beacon 内容会停在 `2412 MHz`。
- 用普通 monitor 后，`survey dump` 可看到 `5745 MHz [in use]`，radiotap 也会变成 `5745 MHz`。
- 因此 OWL 应使用已经建好的普通 monitor，并加 `-N` 跳过内部 active monitor 切换。
- Linux 版 OWL 的 `set_channel()` 需要使用 `NL80211_CHAN_HT20`。如果代码仍是 `NL80211_CHAN_HT40PLUS`，OWL 启动后会把 `monawdl` 从 `149 HT20` 改成 `149 HT40PLUS`，表现为 `center1: 5755 MHz`，iPhone 可能不显示接收端。

```sh
sudo ./build/daemon/owl -i monawdl -h awdl0 -c 149 -v -f -N
```

### AWDL-only UxPlay 验证重点

本轮有效验证方式：

- Avahi 临时限制到 `allow-interfaces=awdl0`，减少普通 LAN 干扰。
- UxPlay 使用独立名称，例如 `OwlPi-P2P149-HT20`。
- `_airplay-p2p._tcp` 手动发布为 `DeviceID@Name` 形式，例如 `B4749F6CBC2F@OwlPi-P2P149-HT20`。
- 同时抓 `awdl0`：

```sh
ip -6 neigh show dev awdl0
avahi-browse -a -t -r
sudo tcpdump -i awdl0 -n -vv 'ip6 and (udp port 5353 or tcp port 7000 or tcp port 7001 or tcp port 7100 or icmp6)'
```

判断：

- 如果 Screen Mirroring 列表不出现，优先查 peer、mDNS、DNS-SD 记录。
- 如果列表出现但点击后没有 TCP 入站，问题还没到 UxPlay，优先查 AirPlay P2P 记录和 AWDL service 兼容性。
- 如果点击后出现 `SYN -> SYN-ACK -> RST`，说明 iPhone 已到达 TCP 资格检查/握手阶段；此时服务名可能被 iPhone 临时隐藏，需换新名字继续测。
- 如果点击后出现 TCP/RTSP 入站，再回到 UxPlay 端口、防火墙、pairing/control 层排查。

## 下一步

1. 固定 channel 149 + 普通 `monawdl` + `owl -N` 作为主线；每次先确认 149 无 `no IR`，再用 `survey dump` 和 radiotap 确认 `5745 MHz`。
2. 以 `OwlPi-P2P149-HT20` 作为稳定发现基线；Mac-like TXT 只用于推进 TCP/RST 诊断，不用于主线提交。
3. 如果 DNS-SD 记录足够接近但仍点击即失败，再分析 OWL 是否缺少新版 iOS 依赖的 `Service Response`、`Service Parameters`、`Bloom Filter` 等 AWDL service TLV 行为。
4. 新购 AR9280 到货后，可重复同样矩阵做 A/B，区分当前 AR928X/转接板组合问题和 OWL/iOS 兼容问题。

## 提交建议

建议主线只提交已经验证有明确作用的内容：

- 文档中的重启复现流程。
- `daemon/netutils.c` 中将 Linux `set_channel()` 固定为 `NL80211_CHAN_HT20`。

暂不建议把实验性的 AirPlay service TLV 注入代码作为稳定版本提交。它能帮助观察和模拟 MacBook 的 AWDL service 行为，但还没有让点击进入 TCP/RTSP；应放在单独实验分支或后续 PR，并标注为 `experimental`。

## 参考

- 本仓库 OWL README：`README.md`
- UxPlay upstream README: https://github.com/FDH2/UxPlay
- OWL upstream README: https://github.com/seemoo-lab/owl
- OpenDrop limitations and compatibility notes: https://deepwiki.com/seemoo-lab/opendrop/1.3-limitations-and-compatibility
- Linux cfg80211 monitor flags documentation: https://docs.kernel.org/driver-api/80211/cfg80211.html
