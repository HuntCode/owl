# AirPlay over OWL/AWDL Validation

Date: 2026-04-28
Last updated: 2026-04-30

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
- Avahi/UxPlay 能发布到 `awdl0 IPv6`，并且 iPhone 的 Screen Mirroring 已能看到 `OwlPi-AWDL` / `OwlPi-AWDL-149P2P`。

### 当前阻塞点

现象：iPhone 能发现接收端，但点击后立即弹出无法连接。

测试中已看到：

- `_airplay._tcp`、`_raop._tcp` 能发布到 `awdl0 IPv6`。
- 手动补充 `_airplay-p2p._tcp`，且服务名改成常见的 `DeviceID@Name` 形式后，iPhone 仍能发现设备。
- `tcpdump -i awdl0` 能看到对端查询 `_airplay-p2p._tcp`，Avahi 能响应 SRV/AAAA/TXT。
- 点击设备失败时，`awdl0` 上没有看到来自 iPhone 的 TCP 7000/7001/7100 入站，UxPlay 也没有 RTSP/pairing 日志。

当前判断：

```text
149/no-IR、真实频道、AWDL peer、IPv6 link-local、mDNS 发现
都不再是当前主阻塞点。

新的阻塞点在 iOS 点击后的 AirPlay P2P 资格检查阶段：
iPhone 在发起 TCP/RTSP 前就判定该目标不可连接。
```

下一层优先看 AirPlay P2P DNS-SD/TXT、`_airplay-p2p._tcp` 记录形态，以及 OWL 对 AWDL service 相关 TLV 的兼容性。

## 关键操作

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

sudo iw dev wlan1 interface add monawdl type monitor flags active
sudo ip link set monawdl up
sudo iw dev monawdl set channel 149 HT20
iw dev monawdl info
sudo tcpdump -i monawdl -e -n -vv 'wlan[0] == 0xd0'
```

通过标准：

- `iw dev monawdl info` 显示 channel 149。
- `tcpdump` radiotap 中出现 `5745 MHz`。
- OWL 日志确认 channel 149 可 frame injection。

作用：

- 排除“看起来设置了 149，实际还在 1/6/36”的误判。
- 确认 iPhone 的 AWDL channel sequence 和 OWL 的实际工作频道能对上。
- 避免把底层频道错误误判成 UxPlay 或 AirPlay 协议错误。

### AWDL-only UxPlay 验证重点

本轮有效验证方式：

- Avahi 临时限制到 `allow-interfaces=awdl0`，减少普通 LAN 干扰。
- UxPlay 使用独立名称，例如 `OwlPi-AWDL-149P2P`。
- `_airplay-p2p._tcp` 手动发布为 `DeviceID@Name` 形式，例如 `B4749F6CBC2F@OwlPi-AWDL-149P2P`。
- 同时抓 `awdl0`：

```sh
ip -6 neigh show dev awdl0
avahi-browse -a -t -r
sudo tcpdump -i awdl0 -n -vv 'ip6 and (udp port 5353 or tcp port 7000 or tcp port 7001 or tcp port 7100 or icmp6)'
```

判断：

- 如果 Screen Mirroring 列表不出现，优先查 peer、mDNS、DNS-SD 记录。
- 如果列表出现但点击后没有 TCP 入站，问题还没到 UxPlay，优先查 AirPlay P2P 记录和 AWDL service 兼容性。
- 如果点击后出现 TCP/RTSP 入站，再回到 UxPlay 端口、防火墙、pairing/control 层排查。

## 下一步

1. 固定 channel 149 + `monawdl` 作为主线；每次先确认 149 无 `no IR`，再用 radiotap 确认 `5745 MHz`。
2. 抓一轮真实 Apple 设备在 AWDL 上发布的 `_airplay._tcp`、`_raop._tcp`、`_airplay-p2p._tcp` TXT 记录，和 UxPlay/Avahi 当前记录逐字段对比。
3. 如果 TXT 差异明显，先用 Avahi 手动发布一组更接近 AppleTV/Mac 的 `_airplay-p2p._tcp` 记录再测。
4. 如果 DNS-SD 记录足够接近但仍点击即失败，再分析 OWL 是否缺少新版 iOS 依赖的 `Service Response`、`Service Parameters`、`Bloom Filter` 等 AWDL service TLV 行为。
5. 新购 AR9280 到货后，可重复同样矩阵做 A/B，区分当前 AR928X/转接板组合问题和 OWL/iOS 兼容问题。

## 参考

- 本仓库 OWL README：`README.md`
- UxPlay upstream README: https://github.com/FDH2/UxPlay
- OWL upstream README: https://github.com/seemoo-lab/owl
- OpenDrop limitations and compatibility notes: https://deepwiki.com/seemoo-lab/opendrop/1.3-limitations-and-compatibility
- Linux cfg80211 monitor flags documentation: https://docs.kernel.org/driver-api/80211/cfg80211.html
